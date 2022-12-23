---
title: "Terraformでcountやfor_eachを使って定義したリソースをimportする方法"
date: 2022-12-06T00:00:00+09:00
lastmod: 2022-12-17T00:00:00+09:00
tags: ["Terraform"]
draft: false
externalUrl: https://qiita.com/chacco38/items/8cc167791633f32c34d9
---

みなさん、こんにちは。今回は既存のリソースを`terraform import`コマンドを使ってTerraform管理下へインポートする際の小ネタの紹介です。

Terraformで実装する際、たとえば複数のリージョンやアベイラビリティゾーンに同様のリソースを配置する際などのユースケースにおいてcountやfor_eachを活用されている方も多いかと思います。このcountやfor_eachを活用して定義したリソースに対して、既存環境を素直にTerraform管理下へインポートしようとすると次のようにコマンドがエラーになってしまいます。今回はそんなときの解決方法を紹介したいと思います。

```txt:エラー時の出力例
$ terraform import google_compute_subnetwork.private["tokyo"] \
    ${GOOGLE_PROJECT}/asia-northeast1/matt-private-snet-tokyo
╷
│ Error: Index value required
│ 
│   on <import-address> line 1:
│    1: google_compute_subnetwork.private[tokyo]
│ 
│ Index brackets must contain either a literal number or a literal string.
╵

For information on valid syntax, see:
https://www.terraform.io/docs/cli/state/resource-addressing.html
```

## countやfor_eachで定義したリソースをimportする方法

結論から言いますと **「'(シングルクォーテーション)で定義したリソース名を囲む」** が1つの解となります。  
出力例のように、Terraformリソース名を'(シングルクォーテーション)で囲んであげると期待通りに既存のリソースがインポートされる動きとなります。めでたしめでたし。

```txt:成功時の出力例
$ terraform import 'google_compute_subnetwork.private["tokyo"]' \
    ${GOOGLE_PROJECT}/asia-northeast1/matt-private-snet-tokyo
Acquiring state lock. This may take a few moments...

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.

Releasing state lock. This may take a few moments...
```

## 余談、ウッカリ違うリソースへimportしてしまった時は

そんなときは一旦ウッカリimportしてしまったリソースをTerraform管理下から外し、正しいリソースへimportしなおしてください。Terraform管理下から外す際の手順は、次の記事を参照していただければと思います。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://qiita.com/chacco38/private/d731a86fcb0626cb958b" frameborder="0" scrolling="no"></iframe>

## 終わりに

いまさらの情報でしたがいかがだったでしょうか。グーグル先生に聞けば一発かも知れませんが、わが身にも起きた事象だったので一応記事にしてみた次第でした。こんな記事でも誰かの手助けになれば幸いです。

以上、既存環境を`terraform import`コマンドを使ってTerraform管理下へインポートする際の小ネタの紹介でした。

---

- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
