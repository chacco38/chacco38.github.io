---
title: "TerraformでAmazon ECSを構築するサンプルコードを書いてみた"
date: 2022-12-29T00:00:00+09:00
lastmod: null
tags: ["Terraform", "AWS", "Amazon ECS", "AWS Fargate", "AWS IAM"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでAmazon ECSのサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

なお、サンプルコードを書いた際のTerraformおよびAWSプロバイダーのバージョンは次のとおりです。最新バージョンでは定義方法が異なっている可能性があるため、実際にコードを書く際は最新の「[Terraformドキュメント]」と「[AWSプロバイダードキュメント]」を確認しながら開発を進めていただければと思います。

[Terraformドキュメント]: https://developer.hashicorp.com/terraform/docs
[AWSプロバイダードキュメント]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

**例）versions.tf**

```tf:versions.tf
# Requirements
terraform {
  required_version = "~> 1.3.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.46.0"
    }
  }
}
```

<!-- omit in toc -->
## サンプルコードを書いてみた

今回は次のような構成のサンプルコードを書いてみました。なお、変数定義部分などの一部省略している点、ならびにステップごとの細かい説明などは省いていますのでご承知おきください。詳細については「[AWSプロバイダードキュメント]」をご参照ください。

- [例1. ECSクラスタの作成](#例1-ecsクラスタの作成)
- [例2. タスク定義の作成](#例2-タスク定義の作成)
- [例3. ECSサービスの作成](#例3-ecsサービスの作成)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

**例）providers.tf**

```tf:providers.tf
# デフォルトのプロバイダー設定
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Owner     = "matt"
      Terraform = "true"
    }
  }
}
```

### 例1. ECSクラスタの作成

ECSクラスタの例でContainer Insightsを有効化している以外には特筆するところはないかと思います。

**例）main.tf**

```tf:main.tf
# ECSクラスタの作成
resource "aws_ecs_cluster" "this" {
  name = var.ecs_cluster_name

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name = var.ecs_cluster_name
  }
}
```

### 例2. タスク定義の作成

AWS Fargate向けのタスク定義の例で、タスク起動用IAMロールやコンテナ用IAMロールなども併せて作成しています。

**例）main.tf**

```tf:main.tf
# タスク定義の作成
resource "aws_ecs_task_definition" "this" {
  family                   = var.ecs_task_name
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.ecs_task_cpu
  memory                   = var.ecs_task_memory
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    # 割愛
  ])

  tags = {
    Name = var.ecs_task_name
  }
}

# タスク起動用IAMロールの定義
resource "aws_iam_role" "ecs_execution" {
  name = var.iam_ecs_execution_role_name

  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "",
        "Effect": "Allow",
        "Principal": {
          "Service": "ecs-tasks.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = var.iam_ecs_execution_role_name
  }
}

# タスク起動用IAMロールへのポリシー割り当て
resource "aws_iam_role_policy_attachment" "ecs_execution" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
  role       = aws_iam_role.ecs_execution.name
}

# コンテナ用IAMロールの定義
resource "aws_iam_role" "ecs_task" {
  name = var.iam_ecs_task_role_name

  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "",
        "Effect": "Allow",
        "Principal": {
          "Service": "ecs-tasks.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = var.iam_ecs_task_role_name
  }
}

# コンテナ用IAMポリシーの定義（例ではSSMパラメータストアのアクセス権限を付与）
resource "aws_iam_policy" "ecs_task" {
  name = var.iam_ecs_task_policy_name
  path = "/service-role/"

  policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": [
          "ssm:GetParametersByPath",
          "ssm:GetParameters",
          "ssm:GetParameter"
        ],
        "Effect": "Allow",
        "Resource": [
          "*"
        ]
      }
    ]
  })

  tags = {
    Name = var.iam_ecs_task_policy_name
  }
}

# コンテナ用IAMロールへのポリシー割り当て
resource "aws_iam_role_policy_attachment" "ecs_task" {
  policy_arn = aws_iam_policy.ecs_task.arn
  role       = aws_iam_role.ecs_task.name
}
```

### 例3. ECSサービスの作成

AWS Fargate向けのECSサービスの例で、タスク起動用IAMロールやコンテナ用IAMロールなども併せて作成しています。

**例）main.tf**

```tf:main.tf
# ECSサービスの作成
resource "aws_ecs_service" "this" {
  name                              = var.ecs_service_name
  cluster                           = aws_ecs_cluster.this.id
  task_definition                   = aws_ecs_task_definition.this.arn
  desired_count                     = var.ecs_desired_count
  health_check_grace_period_seconds = var.ecs_health_check_grace_period_seconds
  launch_type                       = "FARGATE"
  force_new_deployment              = var.ecs_force_new_deployment

  network_configuration {
    security_groups = [aws_security_group.this.id]
    subnets         = [for x in data.aws_subnet.private : x.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.backend.arn
    container_name   = var.ecs_container_name
    container_port   = var.ecs_container_port
  }

  tags = {
    Name = var.ecs_service_name
  }

  depends_on = [aws_lb.cc_external_nlb]
}

# セキュリティグループの定義（通信制御の定義は割愛）
resource "aws_security_group" "this" {
  name   = var.security_group_name
  vpc_id = data.aws_vpc.this.id

  tags = {
    Name = var.security_group_name
  }

  lifecycle {
    create_before_destroy = true
  }
}

# NLB向けのECSサービス用ターゲットグループの定義
resource "aws_lb_target_group" "this" {
  name        = var.nlb_target_group_name
  port        = var.ecs_container_port
  protocol    = "TCP"
  target_type = "ip"
  vpc_id      = data.aws_vpc.this.id

  tags = {
    Name = var.elb_target_group_name
  }
}
```

**例）data.tf**

```tf:data.tf
# VPC情報の取得
data "aws_vpc" "this" {
  cidr_block = var.vpc_cidr_block
}

# サブネット情報の取得
data "aws_subnet" "private" {
  for_each = var.private_subnets

  vpc_id            = data.aws_vpc.default.id
  availability_zone = each.value.availability_zone
  cidr_block        = each.value.cidr_block
}
```

**例）outputs.tf**

```tf:outputs.tf
output "security_group_id" {
  description = "The ID of the security group"
  value       = try(aws_security_group.this.id, "")
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Amazon ECSのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
