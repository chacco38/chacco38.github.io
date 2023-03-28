---
title: "TerraformでAWS Cloud9を構築するサンプルコードを書いてみた"
date: 2022-12-23T00:00:00+09:00
lastmod: null
tags: ["Terraform", "AWS", "AWS Cloud9"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでAWS Cloud9のサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

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

今回は次のような構成のサンプルコードを書いてみました。なお、変数定義部分などの一部省略している点、ならびにステップごとの細かい説明などは省いていますのでご承知おきください。詳細については「[Googleプロバイダードキュメント]」をご参照ください。

- [例1. Cloud9環境の作成](#例1-cloud9環境の作成)
- [例2. Cloud9環境へのメンバー追加](#例2-cloud9環境へのメンバー追加)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

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

### 例1. Cloud9環境の作成

接続方式としてSSMを活用するCloud9環境の例です。automatic_stop_time_minutesパラメータをウッカリ指定し忘れるとEC2インスタンスがずっと起動しっぱなしとなり、ムダに課金をしてしまうことになるためご注意ください。

```tf:main.tf
# Cloud9環境を展開する先のVPC情報の取得
data "aws_vpc" "this" {
  cidr_block = var.vpc_cidr_block

  filter {
    name   = "tag:Name"
    values = [var.vpc_name]
  }
}

# Cloud9環境を展開する先のサブネット情報の取得
data "aws_subnet" "private" {
  vpc_id            = data.aws_vpc.default.id
  availability_zone = var.aws_availability_zone
  cidr_block        = var.private_subnet_cidr_block

  filter {
    name   = "tag:Name"
    values = [var.private_subnet_name]
  }
}

# Cloud9環境の定義
resource "aws_cloud9_environment_ec2" "this" {
  name                        = var.cloud9_name
  instance_type               = "t3.small"
  connection_type             = "CONNECT_SSM"
  subnet_id                   = data.aws_subnet.private.id
  automatic_stop_time_minutes = 30
}
```

### 例2. Cloud9環境へのメンバー追加

本設定は個人でCloud9を立てて使う分には必要はないのですが、だれかと共有して利用するといった場合は次のような形で利用するメンバーを追加で登録することが可能です。

```tf:variables.tf
variable "cloud9_members" {
  descriptions = "IAM user name list of cloud9 environment members"
  default = ["sample_iam_user1", "sample_iam_user2"]
}
```

```tf:main.tf
# Cloud9環境の利用メンバーのIAMユーザの情報取得
data "aws_iam_user" "cloud9_members" {
  for_each = toset(var.cloud9_members)

  user_name = each.value
}

# Cloud9環境を利用するメンバー追加
resource "aws_cloud9_environment_membership" "this" {
  for_each = toset(var.cloud9_members)

  environment_id = aws_cloud9_environment_ec2.this.id
  permissions    = "read-write"
  user_arn       = data.aws_iam_user.cloud9_members[each.value].arn
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Cloud9のサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
