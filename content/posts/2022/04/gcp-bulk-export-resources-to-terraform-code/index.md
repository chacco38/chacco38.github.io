---
title: "Google Cloudに手動で構築した環境からTerraformコードを自動生成する方法"
date: 2022-04-28T00:00:00+09:00
lastmod: null
tags: ["Google Cloud", "Terraform"]
draft: true
externalUrl: null
---

みなさん、こんにちは。昨今、インフラのコード化(IaC)を導入する事例はとても多くなってきていると思いますが、その一方で開発の初期段階からコード化をちゃんとしてだとスピード感が…と悩まれている方もいらっしゃるのではないでしょうか。

今回はそんな悩みに対する1つの解として、開発の初期段階にサクッと手で作った既存のGoogle Cloud環境からTerraformコードを自動生成する方法について紹介していきたいと思います。

## Terraformコードの自動生成方法

いくつか方法はありますが、解の1つは「`gcloud beta resource-config bulk-export`コマンドを使う」です。本コマンドを利用することで既存環境すべてを一括でコード化することも、特定サービスのリソースのみをコード化することも可能です。

なお、コマンドの詳細については公式のリファレンスマニュアルをご参照ください。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://cloud.google.com/sdk/gcloud/reference/beta/resource-config/bulk-export" frameborder="0" scrolling="no"></iframe>

### 既存環境のすべてをコード化する場合の実行例

既存環境のすべてをTerraformコード化する場合は、次のコマンド実行例で

```bash
gcloud beta resource-config bulk-export \
  --resource-format=terraform --path=<出力ディレクトリ>
```

### 特定サービスのリソースのみをコード化する場合の実行例

**実行例）**

```bash
gcloud beta resource-config bulk-export \
  --resource-format=terraform --path=<出力ディレクトリ> \
  --project=<プロジェクト名> --resource-types=<リソース種別名>
```


### 補足、bulk-exportサポート対象リソースの確認方法

**実行例）**

```bash
gcloud beta resource-config list-resource-types --format=yaml
```

**出力例）**

```txt
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

TBD
