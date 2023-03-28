---
title: "TerraformでCloud KMSを構築するサンプルコードを書いてみた"
date: 2022-12-23T00:00:00+09:00
lastmod: 2023-03-26T00:00:00+09:00
tags: ["Terraform", "Google Cloud", "Cloud KMS"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでCloud Key Management Service(KMS)のサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

なお、サンプルコードを書いた際のTerraformおよびGoogleプロバイダーのバージョンは次のとおりです。最新バージョンでは定義方法が異なっている可能性があるため、実際にコードを書く際は最新の「[Terraformドキュメント]」と「[Googleプロバイダードキュメント]」を確認しながら開発を進めていただければと思います。

[Terraformドキュメント]: https://developer.hashicorp.com/terraform/docs
[Googleプロバイダードキュメント]: https://registry.terraform.io/providers/hashicorp/google/latest/docs

```tf:versions.tf
# Requirements
terraform {
  required_version = "1.3.4"

  required_providers {
    google = {
      source  = "hashicorp/google"
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
- [例3. 暗号鍵に対するアクセス権限の付与](#例3-暗号鍵に対するアクセス権限の付与)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

```tf:proiders.tf
# デフォルトのプロバイダー設定
provider "google" {
  project = var.project_id
  region  = "asia-northeast1"
  zone    = "asia-northeast1-b"
}
```

### 例1. キーリングの作成

Cloud Spannerなどのマルチリージョンのリソース向けの鍵を管理するキーリングの例で、今回はasia1(東京/大阪/ソウル)をターゲットにしています。キーリングのオプションはlocationくらいしかないのでとくに困ることはないかと思います。

ただ、キーリングの扱いについては一点注意が必要で、Google Cloudの仕様でGoogle Cloud上からキーリングを削除することはできません。その一方、Terraform管理上はdestroyなどで簡単に削除することができてしまうため、その対処としてTerraform管理上からも削除できないように例ではprevent_destroyを指定します。

```tf:main.tf
# キーリング定義
resource "google_kms_key_ring" "this" {
  name     = "my-kms-keyring-asia1"
  location = "asia1"

  lifecycle {
    prevent_destroy = true
  }
}
```

### 例2. 標準的な顧客管理暗号鍵(CMEK)の作成

Cloud KMS上で顧客管理の暗号鍵を生成する例で、ここでは自動ローテーションを90日(=7,776,000秒)で設定しています。今回のケースではversion_templateの部分はすべてデフォルト値なので、あえて書く必要はなかったのですが明記しておいた方が良いと感じて書きました。

https://cloud.google.com/kms/docs/algorithms

```tf:main.tf
# 暗号鍵定義
resource "google_kms_crypto_key" "this" {
  name            = "my-kms-crypto-key"
  key_ring        = google_kms_key_ring.this.id
  purpose         = "ENCRYPT_DECRYPT"
  rotation_period = "7776000s" # min 86400s(=1d)

  version_template {
    algorithm        = "GOOGLE_SYMMETRIC_ENCRYPTION"
    protection_level = "SOFTWARE"
  }
}
```

### 例3. 暗号鍵に対するアクセス権限の付与

キーポリシーの設定にて対象となる暗号鍵個別に権限を付与する場合の例で、ここでは対象の暗号鍵を使って暗号および複合をできるようにするための権限を特定のサービスアカウントへ付与しています。

余談ですが、権限設定は暗号鍵個別のキーポリシー設定ではなく、IAM設定にて対象アカウントへまるっとCloud KMS権限を付与することでも可能なため、適宜使い分けていただければと思います。

```tf:main.tf
# 権限付与対象のアカウント情報取得（例ではサービスアカウントを指定）
data "google_service_account" "sample" {
  account_id = var.service_account_id
}

# アクセス権の付与
resource "google_kms_crypto_key_iam_binding" "apigee_db" {
  crypto_key_id = google_kms_crypto_key.this.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  members       = [data.google_service_account.this.member]
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Cloud KMSのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- Google Cloud は、Google LLC の商標または登録商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。

