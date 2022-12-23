---
title: "Google Cloudに手動で構築した環境からTerraformコードを自動生成してみた"
date: 2022-04-28T00:00:00+09:00
lastmod: 2022-12-24T00:00:00+09:00
tags: ["Google Cloud", "Terraform"]
draft: false
externalUrl: null
---

みなさん、こんにちは。昨今、インフラのコード化(IaC)を導入する事例はとても多くなってきていると思いますが、その一方で開発の初期段階からしっかりコード化をしようと思うとメンバーのスキルセットによってはスピード感が…と悩まれている方もいらっしゃるのではないでしょうか。

今回はそんな悩みに対する1つの解として、開発の初期段階にサクッと手で作った既存のGoogle Cloud環境からTerraformコードを自動生成する方法について紹介していきたいと思います。

## Terraformコードを自動生成してみた

Terraformコードの自動生成方法としてはいくつか方法はあるのですが、今回はGoogle Cloud CLIの`gcloud beta resource-config bulk-export`コマンドを使ってみました。本コマンドを利用することで既存環境まるっと一括でコード化することも、特定サービスのリソースのみをコード化することも可能です。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://cloud.google.com/sdk/gcloud/reference/beta/resource-config/bulk-export" frameborder="0" scrolling="no"></iframe>

### 例1. 既存環境のすべてをコード化する場合

既存環境のすべてをTerraformコード化する場合は、次のコマンドで自動生成することが可能です。自動生成してみた率直な意見としては、出力されたコード群のディレクトリ構成がサービス毎にバラバラで統一感がないなーでした^^;

```bash:すべてのリソースをコード化する場合
gcloud beta resource-config bulk-export \
  --resource-format=terraform --path=<出力ディレクトリ>
```

### 例2. 特定サービスのリソースのみをコード化する場合

特定のサービスのみをTerraformコード化する場合は、次のコマンドで自動生成することが可能です。なお、複数のリソース種別を「,(カンマ)」でつなげて列挙することでまとめて出力することも可能です。


```bash:特定サービスのリソースをコード化する場合
gcloud beta resource-config bulk-export \
  --resource-format=terraform --path=<出力ディレクトリ> \
  --project=<プロジェクト名> --resource-types=<リソース種別名>
```

なお、`--resource-types`オプションに指定することができるbulk-exportサポート対象のリソース種別名については、次の例のように`gcloud beta resource-config list-resource-types`コマンドで取得することが可能です。

```bash:bulk-exportサポート対象のリソース種別一覧取得
gcloud beta resource-config list-resource-types --format=yaml
```

次の出力例を見ていただくといろいろと出力があるのですが、`--resource-types`オプションに指定する値は`Kind`の部分(=例では`ArtifactRegistryRepository`)となるのでご留意いただければと思います。

```txt:出力例
$ gcloud beta resource-config list-resource-types --format=yaml
:
---
GVK:
  Group: artifactregistry.cnrm.cloud.google.com
  Kind: ArtifactRegistryRepository
  Version: v1beta1
ResourceNameFormat: //artifactregistry.googleapis.com/projects/{{project}}/locations/{{location}}/repositories/{{repository_id}}
SupportsBulkExport: true
SupportsExport: true
SupportsIAM: true
---
:
```

## 終わりに

今回は手動で構築した既存のGoogle Cloud環境からサクッとTerraformコードを自動生成する方法を紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、自動生成されたコードは設定値がハードコーディングされているため、そのままでは他の環境で使いまわせるような代物ではありません。他の環境でも使いまわせるコードを目指すのであれば、べた書きされた設定値を変数化したりなどのリファクタリングが必要となりますのでご注意ください。

とはいえ、リソース定義を書く際の参考としては十分に利用価値はあるかと思いますので、Terraformのコード開発を少しでも楽にする手段の1つとして十分ありなのかなと個人的には思っています^^

---

- Google Cloud は、Google LLC の商標または登録商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。

