---
title: "TerraformでAmazon Route 53を構築するサンプルコードを書いてみた"
date: 2022-12-29T00:00:00+09:00
lastmod: null
tags: ["Terraform", "AWS", "Amazon Route 53"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでAmazon Route 53のサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

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

- [例1. プライベートDNSゾーンの作成](#例1-プライベートdnsゾーンの作成)
- [例2. 同一アカウント内でのプライベートDNSゾーンの共有](#例2-同一アカウント内でのプライベートdnsゾーンの共有)
- [例3. 異なるアカウント間でのプライベートDNSゾーンの共有](#例3-異なるアカウント間でのプライベートdnsゾーンの共有)
- [例4. Aレコードの登録](#例4-aレコードの登録)
- [例5. ALIASレコードの登録](#例5-aliasレコードの登録)
- [例6. CNAMEレコードの登録](#例6-cnameレコードの登録)

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

# プライベートDNSゾーンを共有する先のAWSアカウント情報
provider "aws" {
  alias  = "guest"
  region = var.aws_region
  assume_role {
    role_arn = var.iam_assume_role
  }
  default_tags {
    tags = {
      Owner     = "matt"
      Terraform = "true"
    }
  }
}
```

### 例1. プライベートDNSゾーンの作成

プライベートDNSゾーンの例です。クロスアカウントで共有するといったケースだと`vpc`ブロックにすべて列挙することができないため、後から`aws_route53_zone_association`リソースを作成したことによって生じた差分を無視するように`ignore_changes`を定義しています。

**例）main.tf**

```tf:main.tf
# プライベートDNSゾーンの定義
resource "aws_route53_zone" "this" {
  name = "example.com"

  # NOTE: The aws_route53_zone vpc argument accepts multiple configuration
  #       blocks. The below usage of the single vpc configuration, the
  #       lifecycle configuration, and the aws_route53_zone_association
  #       resource is for illustrative purposes (e.g. for a separate
  #       cross-account authorization process, which is not shown here).
  vpc {
    vpc_id = data.aws_vpc.primary.id
  }

  lifecycle {
    ignore_changes = [vpc]
  }
}
```

**例）data.tf**

```tf:data.tf
# VPC情報の取得
data "aws_vpc" "primary" {
  cidr_block = var.primary_vpc_cidr_block

  filter {
    name   = "tag:Name"
    values = [var.primary_vpc_name]
  }
}
```

### 例2. 同一アカウント内でのプライベートDNSゾーンの共有

同一アカウント内の別VPCからプライベートDNSゾーンに登録された名前を引けるようにする場合の例です。

**例）main.tf**

```tf:main.tf
# 別VPCへのプライベートDNSゾーンの関連付け
resource "aws_route53_zone_association" "this" {
  vpc_id  = data.aws_vpc.secondary.id
  zone_id = aws_route53_zone.this.id
}
```

**例）data.tf**

```tf:data.tf
# VPC情報の取得
data "aws_vpc" "secondary" {
  cidr_block = var.secondary_vpc_cidr_block

  filter {
    name   = "tag:Name"
    values = [var.secondary_vpc_name]
  }
}
```

### 例3. 異なるアカウント間でのプライベートDNSゾーンの共有

異なるアカウントのVPCからプライベートDNSゾーンに登録された名前を引けるようにする場合の例です。

**例）main.tf**

```tf:main.tf
# 別アカウント上のVPCへの割り当て承認
resource "aws_route53_vpc_association_authorization" "this" {
  vpc_id  = data.aws_vpc.guest.id
  zone_id = aws_route53_zone.this.id
}

# 別アカウント上のVPCへのプライベートDNSゾーンの関連付け
resource "aws_route53_zone_association" "this" {
  provider = aws.guest

  vpc_id  = data.aws_vpc.guest.id
  zone_id = aws_route53_zone.this.id
}
```

**例）data.tf**

```tf:data.tf
# VPC情報の取得
data "aws_vpc" "guest" {
  provider = aws.guest

  cidr_block = var.guest_vpc_cidr_block

  filter {
    name   = "tag:Name"
    values = [var.guest_vpc_name]
  }
}
```

### 例4. Aレコードの登録

ドメイン名とIPv4アドレスの対応付けを行うAレコードを登録する場合の例です。

**例）main.tf**

```tf:main.tf
# DNSレコードの定義
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.this.zone_id
  name    = "www.example.com"
  type    = "A"
  ttl     = 300
  records = [aws_eip.alb.public_ip]
}
```

### 例5. ALIASレコードの登録

AWSリソースのオリジナルドメイン名とは異なる名前でアクセスさせたい場合に活用するALIASレコードを登録する場合の例です。なお、ALIASレコードは`ttl`が60秒で固定のためご留意ください。

**例）main.tf**

```tf:main.tf
# DNSレコードの定義
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.this.zone_id
  name    = "www.example.com"
  type    = "A"

  alias {
    name                   = aws_vpc_endpoint.this.dns_entry[0].dns_name
    zone_id                = aws_vpc_endpoint.this.dns_entry[0].hosted_zone_id
    evaluate_target_health = true
  }
}
```

### 例6. CNAMEレコードの登録

オリジナルドメイン名とは異なる名前でアクセスさせたい場合に活用するCNAMEレコードを登録する場合の例です。

**例）main.tf**

```tf:main.tf
# DNSレコードの定義
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.this.zone_id
  name    = "www"
  type    = "CNAME"
  ttl     = 300
  records = ["www.example.co.jp"]
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Amazon Route 53のサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
