---
title: "Terraformでcountやfor_eachを使って定義したリソースをTerraform管理下から外す方法"
date: 2022-12-06T00:00:00+09:00
lastmod: 2022-12-17T00:00:00+09:00
tags: ["Terraform"]
draft: false
externalUrl: https://qiita.com/chacco38/items/d731a86fcb0626cb958b
---

みなさん、こんにちは。今回は既存のリソースを`terraform state rm`コマンドを使ってTerraform管理下から外す際の小ネタの紹介です。

Terraformで実装する際、たとえば複数複数のリージョンやアベイラビリティゾーンに同様のリソースを配置する際などのユースケースにおいてcountやfor_eachを活用されている方も多いかと思います。このcountやfor_eachを活用して定義したリソースに対して素直に`terraform state rm`コマンドを実行すると次のようにコマンドがエラーになってしまいます。今回はそんなときの解決方法を紹介したいと思います。

```txt:エラー時の出力例
$ terraform state rm google_compute_subnetwork.private["tokyo"]
Acquiring state lock. This may take a few moments...
╷
│ Error: Index value required
│ 
│   on  line 1:
│   (source code not available)
│ 
│ Index brackets must contain either a literal number or a literal string.
╵

Releasing state lock. This may take a few moments...
```

## countやfor_eachで定義したリソースを管理外にする方法

結論から言いますと **「'(シングルクォーテーション)で定義したリソース名を囲む」** が1つの解となります。  
出力例のように、Terraformリソース名を'(シングルクォーテーション)で囲んであげると期待通りに既存のリソースがTerraform管理外になる動きとなります。めでたしめでたし。

```txt:成功時の出力例
$ terraform state rm 'google_compute_subnetwork.private["tokyo"]'
Acquiring state lock. This may take a few moments...
Removed google_compute_subnetwork.private["tokyo"]
Successfully removed 1 resource instance(s).
Releasing state lock. This may take a few moments...
```

## 終わりに

いまさらの情報でしたがいかがだったでしょうか。「え、こんなユースケースなくない？」と思われるかもしれませんが、「ついつい、うっかり既存環境を間違って違うリソースにインポートしちゃいました」といったケースもあるかと思います。そんなときの手助けになれば幸いです。

以上、`terraform state rm`コマンドを使ってTerraform管理下から外す際の小ネタの紹介でした。

---

- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
