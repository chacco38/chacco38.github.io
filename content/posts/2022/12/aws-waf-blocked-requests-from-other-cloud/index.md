---
title: "Azure Virtual DesktopからAWSのCloudFrontやAPI Gatewayへのアクセスがブロックされた件について"
date: 2022-12-12T00:00:00+09:00
lastmod: 2022-12-17T00:00:00+09:00
tags: ["AWS", "Amazon CloudFront", "Amazon API Gateway", "AWS WAF", "Azure Virtual Desktop(AVD)"]
draft: false
externalUrl: https://qiita.com/chacco38/items/1a4050454339a559abfd
---

みなさん、こんにちは。AWS WAFを有効化したCloudFrontやAPI Gatewayに対するアクセスがAzure Virtual Desktop(AVD)からだとブロックされてしまうといった事象に遭遇したので、今回はこちらの事象の原因と解決方法について紹介していきたいと思います。

## アクセスがブロックされた原因

WAFのログを確認すると、AWS WAFの匿名IPリストマネージドルールグループ `AWSManagedRulesAnonymousIpList`の`HostingProviderIPList`ルールに合致したためAVDからのアクセスがブロックされていることがわかりました。うーむ、現状だとAzure Virtual Desktopはエンドユーザトラフィックのソースになる可能性が低いと判断されてしまっているということですね…^^;

![01-aws-managed-rule.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/613908/032d52ed-7dc2-f76b-2cd8-d42d9c6654bc.png)

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-ip-rep.html" frameborder="0" scrolling="no"></iframe>

## 解決方法

ホワイトリストを作る(特定のIPアドレスだけ許可する)などのいくつか解決方法はありますが、ホスティングサービスやクラウドサービスを活用して仮想デスクトップを構築・利用しているケースなどもそれなりにいることを考慮すると、このルールで無条件ブロックしていたホスティングサービスやクラウドサービスからの攻撃については別の異なるルールで保護することを前提に、デフォルトアクションを`Block`から`Count`へ変更してしまうという判断もありかと思います。

ということで、次の例のようにデフォルトアクションを変更することで、Azure Virtual Desktopからのアクセスがブロックされないようになりました。めでたしめでたし。

![02-override-action.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/613908/7f72a5c0-e4dd-8539-9083-e5ac8b8baf89.png)

## 余談、Terraformで設定する場合は

managed_rule_group_statementブロック内のexcluded_ruleブロックへ除外したいルールの名前を定義することで実装することが可能です。本定義を利用したサンプルコードを書いてみましたので興味のある方は次の記事をご参照いただければと思います。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://qiita.com/chacco38/private/20553b8967d77028a8c5" frameborder="0" scrolling="no"></iframe>

## 終わりに

今回はたまたまクライアントとしてAzure Virtual Desktopを利用していましたが、マネージドルールの内容を見るかぎり、AzureやGoogle Cloudなどのクラウドサービス全般からのアクセスはことごとくブロックされてしまう可能性があるようにも読めます。もしAWS上のWEBサービスに対して特定ユーザからのアクセスができないようで困っています、といった方は可能性の1つとして同件か疑ってみても良いのかなと思います。

以上、Azure Virtual DesktopからAWSのCloudFrontやAPI Gatewayへのアクセスがブロックされた事象の原因と解決方法のご紹介でした。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Microsoft Azure は，Microsoft Corporation の商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
