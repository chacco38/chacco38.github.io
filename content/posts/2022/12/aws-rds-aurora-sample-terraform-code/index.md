---
title: "TerraformでAmazon Auroraを構築するサンプルコードを書いてみた"
date: 2022-12-23T00:00:00+09:00
lastmod: null
tags: ["Terraform", "AWS", "Amazon Aurora", "Amazon RDS", "AWS IAM"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでAmazon Auroraのサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

なお、サンプルコードを書いた際のTerraformおよびAWSプロバイダーのバージョンは次のとおりです。最新バージョンでは定義方法が異なっている可能性があるため、実際にコードを書く際は最新の「[Terraformドキュメント]」と「[AWSプロバイダードキュメント]」を確認しながら開発を進めていただければと思います。

[Terraformドキュメント]: https://developer.hashicorp.com/terraform/docs
[AWSプロバイダードキュメント]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

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

- [例1. RDSサブネットグループの作成](#例1-rdsサブネットグループの作成)
- [例2. RDSパラメータグループの作成](#例2-rdsパラメータグループの作成)
- [例3. RDSクラスタの作成](#例3-rdsクラスタの作成)
- [例4. RDSインスタンスの作成](#例4-rdsインスタンスの作成)

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

### 例1. RDSサブネットグループの作成

RDSサブネットグループの実装例で、こちらについては特筆すべき点はとくにないと思います。

**例）main.tf**

```tf:main.tf
# RDSサブネットグループの定義
resource "aws_db_subnet_group" "this" {
  name       = var.rds_subnet_group_name
  subnet_ids = [for x in data.aws_subnet.private : x.id]

  tags = {
    Name = var.rds_subnet_group_name
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

### 例2. RDSパラメータグループの作成

RDSパラメータグループの実装例で、定義するパラメータの個数などは環境によってマチマチだと思いますので、変数側で調整できるように`dynamic`ブロックを使って動的に定義しています。

**例）main.tf**

```tf:main.tf
# RDSパラメータグループの定義
resource "aws_rds_cluster_parameter_group" "this" {
  name_prefix = var.rds_parameter_group_name
  family      = var.rds_engine_family

  dynamic "parameter" {
    for_each = var.rds_parameters
    content {
      name         = parameter.value.name
      value        = parameter.value.value
      apply_method = parameter.value.apply_method
    }
  }

  tags = {
    Name = var.rds_parameter_group_name
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

**例）variables.tf**

```tf:variables.tf
# RDSパラメータ一覧
variable "rds_parameters" {
  type    = map(any)
  default = {
    server_audit_events = {
      name         = "server_audit_events"
      value        = "CONNECT,QUERY,TABLE"
      apply_method = "immediate"
    }
    server_audit_logging = {
      name         = "server_audit_logging"
      value        = "1"
      apply_method = "immediate"
    }
    time_zone = {
      name         = "time_zone"
      value        = "Asia/Tokyo"
      apply_method = "pending-reboot"
    }
  }
}
```


### 例3. RDSクラスタの作成

RDSクラスタの実装例で、定義するパラメータの個数などは環境によってマチマチだと思いますので、変数側で調整できるように`dynamic`ブロックを使って動的に定義しています。

**例）main.tf**

```tf:main.tf
# RDSクラスタの定義
resource "aws_rds_cluster" "this" {
  cluster_identifier              = var.rds_cluster_name
  engine                          = var.rds_engine
  engine_version                  = var.rds_engine_version
  database_name                   = var.rds_database_name
  master_username                 = data.aws_ssm_parameter.rds_master_username.value
  master_password                 = data.aws_ssm_parameter.rds_master_password.value
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.this.name

  # storage
  storage_encrypted = true
  kms_key_id        = data.aws_kms_key.this.arn

  # network
  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = [aws_security_group.cc_secret_db.id]
  port                   = var.rds_port

  # monitoring
  enabled_cloudwatch_logs_exports = ["error", "audit", "slowquery"]

  # backup
  backtrack_window          = var.rds_backtrack_window
  backup_retention_period   = var.rds_backup_retention_period
  preferred_backup_window   = var.rds_preferred_backup_window
  copy_tags_to_snapshot     = true
  deletion_protection       = var.rds_deletion_protection
  skip_final_snapshot       = var.rds_skip_final_snapshot
  final_snapshot_identifier = var.rds_skip_final_snapshot ? null : "${var.rds_cluster_name}-snapshot-final"

  # maintenance
  preferred_maintenance_window = var.rds_preferred_maintenance_window
  apply_immediately            = true

  tags = {
    Name = var.rds_cluster_name
  }

  lifecycle {
    ignore_changes = [master_password, availability_zones]
  }
}
```

**例）data.tf**

```tf:data.tf
# VPC情報の取得
data "aws_vpc" "this" {
  cidr_block = var.vpc_cidr_block
}

# SSMパラメータ情報の取得（管理ユーザ名）
data "aws_ssm_parameter" "rds_master_username" {
  name = var.ssm_parameters.rds_master_username.name
}

# SSMパラメータ情報の取得（管理ユーザの初期パスワード）
data "aws_ssm_parameter" "rds_master_password" {
  name = var.ssm_parameters.rds_master_password.name
}

# 暗号鍵情報の取得
data "aws_kms_key" "this" {
  key_id = var.kms_key_alias_name
}
```

### 例4. RDSインスタンスの作成

RDSインスタンスの実装例で、モニタリング用のIAMロールの定義なども併せて記載しています。

なお、マスターユーザ/パスワード情報はSecrets Managerで管理というケースが最近は多くなってきていると思いますが、今回の例ではパスワード情報はプロジェクト管理とし、ここではあくまで構築用の初期パスワードという扱いでSSMパラメータストアにSecureStingとして事前に格納しておいた情報を扱うようにしています。

**例）main.tf**

```tf:main.tf
# RDSインスタンスの定義
resource "aws_rds_cluster_instance" "this" {
  count = var.rds_instance_count

  cluster_identifier = aws_rds_cluster.this.id
  identifier         = "${var.rds_instance_name}-${count.index + 1}"
  instance_class     = var.rds_instance_class
  engine             = aws_rds_cluster.this.engine
  engine_version     = aws_rds_cluster.this.engine_version

  # network
  db_subnet_group_name = aws_db_subnet_group.this.name

  # monitoring
  monitoring_interval                   = var.rds_monitoring_interval
  monitoring_role_arn                   = aws_iam_role.monitoring.arn
  performance_insights_enabled          = true
  performance_insights_kms_key_id       = data.aws_kms_key.this.arn
  performance_insights_retention_period = var.rds_performance_insights_retention_period

  # backup
  copy_tags_to_snapshot = true

  # maintenance
  preferred_maintenance_window = var.rds_preferred_maintenance_window
  auto_minor_version_upgrade   = false
  publicly_accessible          = false
  apply_immediately            = true

  tags = {
    Name = var.rds_instance_name
  }
}

# モニタリング用のIAMロール定義
resource "aws_iam_role" "monitoring" {
  name = var.monitoring_role_name

  assume_role_policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Sid" : "",
        "Effect" : "Allow",
        "Principal" : {
          "Service" : "monitoring.rds.amazonaws.com"
        },
        "Action" : "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = monitoring_role_name
  }
}

# モニタリング用のIAMロールへのポリシー追加
resource "aws_iam_role_policy_attachment" "this" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
  role       = aws_iam_role.monitoring.name
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Amazon Auroraのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
