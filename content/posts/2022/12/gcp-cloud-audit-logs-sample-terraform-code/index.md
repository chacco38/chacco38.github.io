---
title: "TerraformでCloud Audit Logsを構築するサンプルコードを書いてみた"
date: 2022-12-26T00:00:00+09:00
lastmod: null
tags: ["Terraform", "Google Cloud", "Cloud Audit Logs"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでCloud Audit Logsのサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

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

- [例1. データアクセス監査ログのデフォルト構成設定](#例1-データアクセス監査ログのデフォルト構成設定)
- [例2. サービスごとのデータアクセス監査ログ構成設定](#例2-サービスごとのデータアクセス監査ログ構成設定)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

```tf:proiders.tf
# デフォルトのプロバイダー設定
provider "google" {
  project = var.project_id
  region  = "asia-northeast1"
  zone    = "asia-northeast1-b"
}
```

### 例1. データアクセス監査ログのデフォルト構成設定

データアクセス監査ログのデフォルト構成にて、すべてのログタイプ(=`ADMIN_READ`、`DATA_READ`、`DATA_WRITE`の3種)を有効化する場合の例です。コード量を減らすためにdynamicブロックを使ってaudit_log_configブロックを定義してみました。

```tf:main.tf
# デフォルトで有効化するログタイプ一覧
variable "default_enabled_log_types" {
  default = ["ADMIN_READ", "DATA_READ", "DATA_WRITE"]
}

# 設定対象プロジェクトの情報取得
data "google_project" "this" {}

# Cloud Audit Logsリソース定義
resource "google_project_iam_audit_config" "default" {
  count = length(var.default_enabled_log_types) > 0 ? 1 : 0

  project = data.google_project.this.id
  service = "allServices"

  dynamic "audit_log_config" {
    for_each = var.default_enabled_log_types
    content {
      log_type = audit_log_config.value
    }
  }
}
```

### 例2. サービスごとのデータアクセス監査ログ構成設定

特定のサービスのデータアクセス監査ログを有効化する場合の例です。今回はCloud SQLとCloud Spannerのデータ書き込み`DATA_WRITE`監査ログのみを有効化しています。なお、Cloud Audit Logsの設定で指定するサービス名については公式ドキュメントの「[サービスとリソースのマッピング](https://cloud.google.com/logging/docs/api/v2/resource-list?hl=ja#service-names)」を参照してください。

```tf:main.tf
# サービスごとに有効化するログタイプ一覧
variable "enabled_log_types" {
  default = {
    "cloudsql.googleapis.com" = ["DATA_WRITE"]
    "spanner.googleapis.com"  = ["DATA_WRITE"]
  }
}

# 設定対象プロジェクトの情報取得
data "google_project" "this" {}

# Cloud Audit Logsリソース定義
resource "google_project_iam_audit_config" "service" {
  for_each = var.enabled_log_types

  project = data.google_project.this.id
  service = each.key

  dynamic "audit_log_config" {
    for_each = each.value
    content {
      log_type = audit_log_config.value
    }
  }
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Cloud Audit Logsのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。

記事執筆時点では`google_project_iam_audit_config`リソースはaudit_log_configブロックが1つ以上定義されていないとエラーとなる仕様のため、今回は有効化するログがなければそもそもリソースを作らないようにしています。ただ、これだとリソースを作ってないことになるので、あとから手で有効化された際にTerraformで差分を検知できません。そこの点が正直イケてないなぁと思ってますが、こんな記事でも誰かの役に立っていただけるのであれば幸いです^^;

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- Google Cloud は、Google LLC の商標または登録商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。

