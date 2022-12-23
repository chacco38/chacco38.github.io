---
title: "Terraformで管理するAWSリソースへタグ情報を付与する際の小ネタ紹介"
date: 2022-12-13T00:00:00+09:00
lastmod: 2022-12-17T00:00:00+09:00
tags: ["AWS", "Terraform"]
draft: false
externalUrl: https://qiita.com/chacco38/items/6835f0673420d6b6a623
---

みなさん、こんにちは。AWSを利用するにあたってタグの付与は、運用管理を容易にするなどの観点から一般的に行われている設定かと思います。今回はTerraform入門としてそんなAWSタグをTerraformで付与する際の小ネタのご紹介です。

## 基本的なタグの設定方法

基本的な設定方法としてはAWSプロバイダーの定義にて、デフォルトで付与するタグを設定することです。これにより、すべてのAWSリソースへ漏れることなく付与することが可能なのでこちらを活用するのがオススメです。

```tf
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      owner     = "matt"
      terraform = "true"
    }
  }
}
```

## 複数パターンのタグ設定方法

「[基本的なタグの設定方法](#基本的なタグの設定方法)」では1つのパターンのみでしたが、たとえば環境を表すタグ情報などをちょっと変えたものをいくつかグルーピングしたい、といったケースもあるかと思います。そのような場合はグループごとにプロバイダー定義を用意して、各リソース定義にてプロバイダーを指定することによって期待通りの動作をさせることができます。

```tf
provider "aws" {
  alias  = "dev"
  region = var.aws_region
  default_tags {
    tags = {
      owner       = "matt"
      environment = "dev"
      terraform   = "true"
    }
  }
}

provider "aws" {
  alias  = "prod"
  region = var.aws_region
  default_tags {
    tags = {
      owner       = "matt"
      environmemt = "prod"
      terraform   = "true"
    }
  }
}

resource "aws_xxxx_xxxx" "dev" {
  provider = aws.dev
  # 以下、省略
}

resource "aws_xxxx_xxxx" "prod" {
  provider = aws.prod
  # 以下、省略
}
```

## リソース個別タグの設定方法

リソースの名前タグなどのリソースごとに異なる値となるタグについては、各リソース定義のtagsブロック内で定義します。なお、プロバイダー設定で行ったタグ設定と同じキーのタグを、各リソース定義のtagsブロックにて指定することによって値を上書きすることも可能です。

```tf
resource "aws_xxxx_xxxx" "sample" {
  name = var.resource_name
  # 途中、省略
  tags = {
    name = var.resource_name
  }
}
```

## モジュールへのタグ設定の継承方法

モジュールへタグ設定を継承する場合は、providersブロックを用いて継承したいタグ設定を行ったプロバイダーを指定することで可能です。なお、インターネット上にはモジュール側に空のプロバイダーブロックを継承したプロバイダーの分定義するといったサンプルもありますが、そちらは古いフォーマットなのでご注意ください。

```tf:メイン定義
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      owner      = "matt"
      workload   = "common"
      terraform  = "true"
    }
  }
}

provider "aws" {
  alias  = "web"
  region = var.aws_region
  default_tags {
    tags = {
      owner      = "matt"
      workload   = "web"
      terraform  = "true"
    }
  }
}

module "sample" {
  source "../../modules/sample"
  providers = {
    aws     = aws
    aws.web = aws.web
  }
  # 以下、省略
}
```

```tf:モジュール定義
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      configuration_aliases = [
        aws,
        aws.web,
      ]
    }
  }
}

resource "aws_xxxx_xxxx" "default" {
  # 以下、省略
}

resource "aws_xxxx_xxxx" "web" {
  provider = aws.web
  # 以下、省略
}
```

<!-- omit in toc -->
## 終わりに

普段からTerraformを触っている人からすると「なにをいまさら…」といった情報かもしれませんが、こんな記事でもだれかの助けになれれば幸いです。以上、Terraformで管理するAWSリソースへタグ情報を付与する際の小ネタの紹介でした。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。

