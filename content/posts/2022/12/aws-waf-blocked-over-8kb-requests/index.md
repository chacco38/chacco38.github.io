---
title: "AWS WAFを有効化したCloudFrontやAPI Gatewayに対する8KBを超えるリクエストに失敗した件について"
date: 2022-12-12T00:00:00+09:00
lastmod: 2022-12-17T00:00:00+09:00
tags: ["AWS", "Amazon CloudFront", "Amazon API Gateway", "AWS WAF"]
draft: false
externalUrl: https://qiita.com/chacco38/items/1389102d7b8f95a5deb2
---

みなさん、こんにちは。AWS WAFを有効化したCloudFrontやAPI Gatewayに対する8KB以上のリクエストがブロックされてしまうといった事象に遭遇したので、今回はこちらの事象の原因と解決方法について紹介していきたいと思います。

## リクエストがブロックされた原因

WAFのログを確認すると、そのままズバリAWS WAFのコアルールセット(CRS)マネージドルールグループ `AWSManagedRulesCommonRuleSet`の`SizeRestrictions_BODY`ルールに合致したため8KB(8,192バイト)を超えるBodyを持つリクエストがブロックされていることがわかりました。

余談ですが、なぜ8KBという制限が存在するかというとそもそもAWS WAFで検査できるBodyは最初の8KBという制限があります(おそらく性能の観点で)。そのため、8KBを超えるBodyを持つ(=全文検査ができない)リクエストをデフォルトでブロックしているのではないか、と個人的には考えております。

![01-aws-managed-rule.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/613908/9c4ec5d9-4ac8-e879-05b7-64bea234ffdc.png)

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-baseline.html" frameborder="0" scrolling="no"></iframe>

## 解決方法

次の例のように`SizeRestrictions_BODY`ルールのデフォルトアクションを`Block`から`Count`へ変更することによって、8KBを超えるBodyを持つリクエストがブロックされず、ちゃんと扱えるようになりました。めでたしめでたし。

![02-override-action.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/613908/3817eaee-9581-fbca-8279-8cbf32ab75cb.png)

## 余談、Terraformで設定する場合は

managed_rule_group_statementブロック内のexcluded_ruleブロックへ除外したいルールの名前を定義することで実装することが可能です。本定義を利用したサンプルコードを書いてみましたので興味のある方は次の記事をご参照いただければと思います。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://qiita.com/chacco38/private/20553b8967d77028a8c5" frameborder="0" scrolling="no"></iframe>

## 終わりに

API Gatewayを扱ってきた方からするとなにを今更いってんだか…といった内容かもしれませんが、こんな情報でも誰かの役に経っていただければ幸いです。なお、今回ご紹介したマネージドルールを抜きにしても、AWS WAFにはBody以外にもHeaderやCookieにもサイズや数の制限がありますので、それを超えるリクエストを扱う場合は次のサイトを参考にしていただければと思います。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/waf-rule-statement-oversize-handling.html" frameborder="0" scrolling="no"></iframe>

以上、AWS WAFを有効化したCloudFrontやAPI Gatewayに対する8KBを超えるリクエストがブロックされた事象の原因と解決方法のご紹介でした。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
