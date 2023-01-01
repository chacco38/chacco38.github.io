---
title: "TerraformでAWS Cloud9を構築するサンプルコードを書いてみた"
date: 2022-12-23T00:00:00+09:00
lastmod: null
tags: ["Terraform", "AWS", "AWS Cloud9"]
draft: true
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでAWS Cloud9のサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

なお、サンプルコードを書いた際のTerraformおよびAWSプロバイダーのバージョンは次のとおりです。最新バージョンでは定義方法が異なっている可能性があるため、実際にコードを書く際は最新の「[Terraformドキュメント]」と「[AWSプロバイダードキュメント]」を確認しながら開発を進めていただければと思います。

[Terraformドキュメント]: https://developer.hashicorp.com/terraform/docs
[AWSプロバイダードキュメント]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

```tf:versions.tf
# Requirements
terraform {
  required_version = "1.3.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.46.0"
    }
  }
}
```

<!-- omit in toc -->
## サンプルコードを書いてみた

今回は次のような構成のサンプルコードを書いてみました。なお、変数定義部分などの一部省略している点、ならびにステップごとの細かい説明などは省いていますのでご承知おきください。詳細については「[Googleプロバイダードキュメント]」をご参照ください。

- [例1. キーリングの作成](#例1-キーリングの作成)
- [例2. 標準的な顧客管理暗号鍵(CMEK)の作成](#例2-標準的な顧客管理暗号鍵cmekの作成)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

```tf:proiders.tf
# デフォルトのプロバイダー設定
provider "google" {
  project = var.project_id
  region  = "asia-northeast1"
  zone    = "asia-northeast1-b"
}
```

### 例1. Cloud9環境の作成

```tf
resource "google_kms_key_ring" "sample" {
  name     = "my-kms-keyring-asia1"
  location = "asia1"

  lifecycle {
    prevent_destroy = true
  }
}
```

### 例2. 標準的な顧客管理暗号鍵(CMEK)の作成

Cloud KMS上で顧客管理の暗号鍵を生成する例で、自動ローテーションを90日(=7,776,000秒)で設定しています。鍵の目的については利用用途に寄って

<https://cloud.google.com/kms/docs/algorithms>

```
resource "google_kms_crypto_key" "sample" {
  name            = "my-kms-crypto-key"
  key_ring        = google_kms_key_ring.sample.id
  purpose         = "ENCRYPT_DECRYPT"
  rotation_period = "7776000s" # min 86400s(=1d)
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Cloud KMSのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
