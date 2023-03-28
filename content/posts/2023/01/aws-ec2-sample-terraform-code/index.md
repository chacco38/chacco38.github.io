---
title: "TerraformでAmazon EC2を構築するサンプルコードを書いてみた"
date: 2023-01-04T00:00:00+09:00
lastmod: null
tags: ["Terraform", "AWS", "Amazon EC2"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでAmazon EC2のサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

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

- [例1. シングルEC2インスタンスの作成](#例1-シングルec2インスタンスの作成)

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

### 例1. シングルEC2インスタンスの作成

KMSで管理する暗号鍵にてディスク暗号化を施したEC2インスタンスを実装する際の例です。追加ディスクの有無や個数は、サーバ要件によって大きく異なると思うので`dynamic`ブロックを使って動的定義にしています。

**例）main.tf**

```tf:main.tf
# Amazon EC2インスタンスの定義
resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type

  availability_zone = var.availability_zone
  subnet_id         = data.aws_subnet.this.id

  vpc_security_group_ids = [aws_security_group.this.id]
  monitoring             = true

  root_block_device {
    volume_size = lookup(var.root_block_device, "volume_size", null)
    volume_type = lookup(var.root_block_device, "volume_type", null)
    iops        = lookup(var.root_block_device, "iops", null)
    throughput  = lookup(var.root_block_device, "throughput", null)
    encrypted   = true
    kms_key_id  = data.aws_kms_key.this.arn
  }

  dynamic "ebs_block_device" {
    for_each = var.ebs_block_devices
    content {
      device_name = lookup(ebs_block_device.value, "device_name", null)
      volume_size = lookup(ebs_block_device.value, "volume_size", null)
      volume_type = lookup(ebs_block_device.value, "volume_type", null)
      iops        = lookup(ebs_block_device.value, "iops", null)
      throughput  = lookup(ebs_block_device.value, "throughput", null)
      snapshot_id = lookup(ebs_block_device.value, "snapshot_id", null)
      encrypted   = true
      kms_key_id  = data.aws_kms_key.this.arn
    }
  }

  tags = {
    Name = var.instance_name
  }
}

# 空のセキュリティグループの作成
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
```

**例）data.tf**

```tf:data.tf
# VPC情報の取得
data "aws_vpc" "this" {
  cidr_block = var.vpc_cidr_block

  filter {
    name   = "tag:Name"
    values = [var.vpc_name]
  }
}

# サブネット情報の取得
data "aws_subnet" "private" {
  vpc_id            = data.aws_vpc.default.id
  availability_zone = var.availability_zone
  cidr_block        = var.private_subnet_cidr_block

  filter {
    name   = "tag:Name"
    values = [var.private_subnet_name]
  }
}

# 暗号鍵情報の取得
data "aws_kms_key" "this" {
  key_id = var.kms_key_alias_name
}
```

**例）outputs.tf**

```tf:outputs.tf
output "instance_id" {
  description = "The ID of the instance"
  value       = try(aws_instance.this.id, "")
}

output "instance_arn" {
  description = "The ARN of the instance"
  value       = try(aws_instance.this.arn, "")
}

output "instance_private_ip" {
  description = "The private IP address of the instance"
  value       = try(aws_instance.this.private_ip, "")
}

output "security_group_id" {
  description = "The ID of the security group"
  value       = try(aws_security_group.this.id, "")
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Amazon EC2のサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
