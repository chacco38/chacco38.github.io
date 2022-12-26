---
title: "TerraformでCloud Spannerを構築するサンプルコードを書いてみた"
date: 2022-04-28T00:00:00+09:00
lastmod: 2022-12-23T00:00:00+09:00
tags: ["Google Cloud", "Cloud Spanner", "Terraform"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでCloud Spannerのサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

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

- [例1. 東京/大阪マルチリージョンインスタンスの例](#例1-東京大阪マルチリージョンインスタンスの例)
- [例2. Google Standard SQL言語データベースの例](#例2-google-standard-sql言語データベースの例)
- [例3. PostgreSQL言語データベースの例](#例3-postgresql言語データベースの例)
- [例4. 顧客管理暗号化鍵(CMEK)を利用したデータベースの例](#例4-顧客管理暗号化鍵cmekを利用したデータベースの例)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

```tf:providers.tf
# デフォルトのプロバイダー設定
provider "google" {
  project = var.project_id
  region  = "asia-northeast1"
  zone    = "asia-northeast1-b"
}
```

### 例1. 東京/大阪マルチリージョンインスタンスの例

Cloud Spannerのマルチリージョンインスタンスの例です。なお、コンピューティング容量の割り当て単位としては、今回の例で採用している処理ユニットベース以外にノード単位でも指定できますが、どちらで指定しても結果に大差はないのでノード単位の例は割愛します。

```tf:main.tf
# asia1マルチリージョンのSpannerインスタンス定義
resource "google_spanner_instance" "sample" {
  display_name     = "my-sample-spanner" # 4-30 characters
  config           = "asia1"
  processing_units = 100                 # 1000 == 1 node
  force_destroy    = var.force_destry_enabled ? true : false
}
```

### 例2. Google Standard SQL言語データベースの例

Google Standard SQL言語ベースのデータベースを作成する例です。`database_dialect`オプションはデフォルト値が`GOOGLE_STANDARD_SQL`なので省略可能なのですが、個人的にはここの部分は明示したい気持ちに駆られるのであえて書いてます^^; `version_retention_period`オプションについても同様で、要件にあわせて変える部分だと思うのでここもあえて書いてみました。

```tf:main.tf
# Google Standard SQL言語ベースのデータベース定義
resource "google_spanner_database" "google" {
  instance                 = google_spanner_instance.sample.id
  name                     = "my-googlesql-db" # 2-30 characters
  database_dialect         = "GOOGLE_STANDARD_SQL"
  version_retention_period = "3d" # Default 1h, Max 7d
  deletion_protection      = var.deletion_protection_enabled ? true : false
}
```

### 例3. PostgreSQL言語データベースの例

PostgreSQL言語ベースのデータベースを作成する例です。「[例2](#例2-google-standard-sql言語データベースの例)」とほぼ同じなので特筆すべき点はないかと思います。


```tf:main.tf
# PostgreSQL言語ベースのデータベース定義
resource "google_spanner_database" "postgre" {
  instance                 = google_spanner_instance.sample.id
  name                     = "my-postgresql-db" # 2-30 characters
  database_dialect         = "POSTGRESQL"
  version_retention_period = "24h" # Default 1h, Max 7d
  deletion_protection      = var.deletion_protection_enabled ? true : false
}
```

### 例4. 顧客管理暗号化鍵(CMEK)を利用したデータベースの例

法令順守などのために顧客が管理する暗号鍵を利用したデータベースを作成する際の例です。dataブロック定義を使って作成済みのCloud KMSリソースのIDを取得し、それを指定するようにしています。なお、今回はCloud Spanner用サービスアカウントに特定の暗号化鍵に対して個別にアクセス権を付与するようにしています。

```tf:main.tf
# Cloud KMSで管理する顧客管理暗号鍵(CMEK)で暗号化したデータベース定義
resource "google_spanner_database" "google" {
  instance                 = google_spanner_instance.sample[0].name
  name                     = "my-cmek-encrypted-db" # 2-30 characters
  database_dialect         = "GOOGLE_STANDARD_SQL"
  version_retention_period = "3d"

  encryption_config {
    kms_key_name = data.google_kms_crypto_key.spanner.id
  }

  deletion_protection = var.deletion_protection_enabled ? true : false
}

# Cloud KMSで管理する顧客管理暗号鍵(CMEK)を利用するための権限付与
resource "google_kms_crypto_key_iam_member" "crypto_key" {
  crypto_key_id = data.google_kms_crypto_key.spanner.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  member        = "serviceAccount:service-${data.google_project.this.number}@gcp-sa-spanner.iam.gserviceaccount.com"
}

# プロジェクトリソース取得
data "google_project" "this" {}

# キーリングリソース取得
data "google_kms_key_ring" "asia1" {
  name     = var.gcp_kms_keyring_name
  location = "asia1"
}

# 暗号化キーリソース取得
data "google_kms_crypto_key" "spanner" {
  name     = var.gcp_kms_key_nameGoogle
  key_ring = data.google_kms_key_ring.asia1.id
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Cloud Spannerのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- Google Cloud は、Google LLC の商標または登録商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。

