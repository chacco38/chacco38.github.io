---
title: "TerraformでAmazon RDS for Oracleを構築するサンプルコードを書いてみた"
date: 2023-01-04T00:00:00+09:00
lastmod: 2023-03-26T00:00:00+09:00
tags: ["Terraform", "AWS", "Amazon RDS", "Amazon RDS for Oracle", "AWS IAM", "Security Group"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでAmazon RDS for Oracleのサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

なお、サンプルコードを書いた際のTerraformおよびAWSプロバイダーのバージョンは次のとおりです。最新バージョンでは定義方法が異なっている可能性があるため、実際にコードを書く際は最新の「[Terraformドキュメント]」と「[AWSプロバイダードキュメント]」を確認しながら開発を進めていただければと思います。

[Terraformドキュメント]: https://developer.hashicorp.com/terraform/docs
[AWSプロバイダードキュメント]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

**例）versions.tf**

```tf:versions.tf {linenos=table}
# Requirements
terraform {
  required_version = "~> 1.3.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.55.0"
    }
  }
}
```

<!-- omit in toc -->
## サンプルコードを書いてみた

今回は次のような構成のサンプルコードを書いてみました。なお、変数定義部分などの一部省略している点、ならびにステップごとの細かい説明などは省いていますのでご承知おきください。詳細については「[AWSプロバイダードキュメント]」をご参照ください。

- [例1. RDSサブネットグループの作成](#例1-rdsサブネットグループの作成)
- [例2. RDSパラメータグループの作成](#例2-rdsパラメータグループの作成)
- [例3. RDSオプショングループの作成](#例3-rdsオプショングループの作成)
- [例4. RDSインスタンスの作成](#例4-rdsインスタンスの作成)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

**例）providers.tf**

```tf:providers.tf {linenos=table}
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

```tf:main.tf {linenos=table}
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

```tf:data.tf {linenos=table}
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

```tf:main.tf {linenos=table}
# RDSパラメータグループの定義
resource "aws_db_parameter_group" "this" {
  name   = var.rds_parameter_group_name
  family = var.rds_engine_family

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
}
```

**例）variables.tf**

```tf:variables.tf {linenos=table}
# RDSパラメータ一覧
variable "rds_parameters" {
  type    = map(any)
  default = {
    audit_trail = {
      name         = "audit_trail"
      value        = "DB,EXTENDED"
      apply_method = "pending-reboot"
    }
  }
}
```

### 例3. RDSオプショングループの作成

RDSオプショングループの実装例で、タイムゾーンの設定とS3統合の設定だけしています。S3統合を有効にしているのは、なんとなくDATA PUMPを使って手軽にエクスポート/インポートをしたいケースもあるかな、と思って有効にしています。

**例）main.tf**

```tf:main.tf {linenos=table}
# RDSオプショングループの定義
resource "aws_db_option_group" "this" {
  name                 = var.rds_option_group_name
  engine_name          = var.rds_engine
  major_engine_version = var.rds_engine_major_version

  option {
    option_name = "Timezone"

    option_settings {
      name  = "TIME_ZONE"
      value = "Asia/Tokyo"
    }
  }

  option {
    option_name = "S3_INTEGRATION"
    version     = "1.0"
  }

  tags = {
    Name = var.rds_option_group_name
  }
}
```

### 例4. RDSインスタンスの作成

RDSインスタンスの実装例で、モニタリング用やS3統合用のIAMロールの定義なども併せて記載しています。

なお、マスターユーザ/パスワード情報はSecrets Managerで管理というケースが最近は多くなってきていると思いますが、今回の例ではパスワード情報はプロジェクト管理とし、ここではあくまで構築用の初期パスワードという扱いでSSMパラメータストアにSecureStingとして事前に格納しておいた情報を扱うようにしています。

**例）main.tf**

```tf:main.tf {linenos=table}
# RDSインスタンスの定義
resource "aws_db_instance" "this" {
  identifier               = var.rds_instance_name
  instance_class           = var.rds_instance_class
  engine                   = var.rds_engine
  engine_version           = var.rds_engine_version
  license_model            = "license-included"
  multi_az                 = var.rds_malti_az
  username                 = data.aws_ssm_parameter.rds_master_username.value
  password                 = data.aws_ssm_parameter.rds_master_password.value
  parameter_group_name     = aws_db_parameter_group.this.name
  option_group_name        = aws_db_option_group.this.name

  # storage
  storage_type          = var.rds_storage_type
  allocated_storage     = var.rds_allocated_storage
  max_allocated_storage = var.rds_max_allocated_storage
  storage_encrypted     = true
  kms_key_id            = data.aws_kms_key.this.arn

  # network
  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = [aws_security_group.this.id]
  port                   = var.rds_port

  # monitoring
  monitoring_interval                   = var.rds_monitoring_interval
  monitoring_role_arn                   = aws_iam_role.monitoring.arn
  performance_insights_enabled          = true
  performance_insights_kms_key_id       = data.aws_kms_key.this.arn
  performance_insights_retention_period = var.rds_performance_insights_retention_period

  # backup
  backup_retention_period  = var.rds_backup_retention_period
  backup_window            = var.rds_backup_window
  copy_tags_to_snapshot    = true
  delete_automated_backups = true
  deletion_protection      = false
  skip_final_snapshot      = true

  # maintenance
  maintenance_window         = var.rds_maintenance_windows
  auto_minor_version_upgrade = false
  publicly_accessible        = false
  apply_immediately          = true

  lifecycle {
    ignore_changes = [password]
  }

  tags = {
    Name = var.rds_instance_name
  }
}

# 空のセキュリティグループの定義
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

# S3統合用のIAMロール定義
resource "aws_iam_role" "s3_integration" {
  name = var.iam_s3integration_role_name

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
    Name = var.iam_s3integration_role_name
  }
}

# S3統合用のIAMポリシー定義
resource "aws_iam_policy" "s3integration" {
  name = var.iam_s3integration_policy_name
  path = "/"

  policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:ListBucket",
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:PutObjectAcl"
        ],
        "Resource": [
          "${data.aws_s3_bucket.s3_integration.arn}",
          "${data.aws_s3_bucket.s3_integration.arn}/*"
        ]
      },
      {
        "Effect": "Allow",
        "Action": [
          "kms:Encrypt",
          "kms:ReEncrypt*",
          "kms:Decrypt",
          "kms:DescribeKey",
          "kms:GenerateDataKey"
        ],
        "Resource": [
          "${data.aws_kms_key.this.arn}"
        ]
      }
    ]
  })

  tags = {
    Name = var.iam_s3integration_policy_name
  }
}

# S3統合用IAMロールにIAMポリシーを割り当て
resource "aws_iam_role_policy_attachment" "s3_integration" {
  policy_arn = aws_iam_policy.s3_integration.arn
  role       = aws_iam_role.s3_integration.name
}

# RDBインスタンスにS3統合用のIAMロールの割り当て
resource "aws_db_instance_role_association" "s3_integration" {
  db_instance_identifier = aws_db_instance.this.id
  feature_name           = "S3_INTEGRATION"
  role_arn               = aws_iam_role.s3_integration.arn
}
```

**例）data.tf**

```tf:data.tf {linenos=table}
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

# S3統合用S3バケット情報の取得
data "aws_s3_bucket" "s3_integration" {
  bucket = var.s3_bucket_name
}
```

**例）outputs.tf**

```tf:outputs.tf {linenos=table}
output "security_group_id" {
  description = "The ID of the security group"
  value       = try(aws_security_group.this.id, "")
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Amazon RDS for Oracleのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
