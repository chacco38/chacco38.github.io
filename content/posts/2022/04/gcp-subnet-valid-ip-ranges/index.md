---
title: "Google Cloudのサブネットには有効なIPアドレス範囲に制限があるのでご注意を"
date: 2022-04-12T00:00:00+09:00
lastmod: null
tags: ["Google Cloud", "VPC", "VPC Peering"]
draft: false
externalUrl: null
---

みなさん、こんにちは。Google Cloudのサブネットにはソフトウェア的に割り当てを制限されてはいないものの、実際に割り当ててしまうと通信障害などを起こしてしまう割り当てはいけないIPv4アドレス範囲があるのをご存じでしょうか。

今回は意外と見落としがちなGoogle Cloudのサブネットで有効なIPv4アドレス範囲、および範囲外のアドレスを割り当ててしまった際にはこんな事象が起きましたよ、という実体験について紹介していきたいと思います。

## 有効なIPv4アドレス範囲とは

それでは早速ですが、次の一覧（[公式ドキュメント]より抜粋）が有効なIPv4アドレス範囲の答えとなっています。なお、特段の要件がない限りは、基本的に★マークをつけた「[RFC1918]」に準拠する形でサブネットの設計をしていただくのが無難かと思います。

[公式ドキュメント]: https://cloud.google.com/vpc/docs/subnets?hl=ja#valid-ranges

||有効なIPv4アドレス範囲|ソース|説明|
|:---:|:---|:---|:---|
|★|**10.0.0.0/8<br>172.16.0.0/12<br>192.168.0.0/16**|**[RFC1918]**|**プライベートIPアドレス**|
||100.64.0.0/10|[RFC6598]|共有アドレス空間|
||192.0.0.0/24|[RFC6890]|IETF プロトコルの割り当て|
||192.0.2.0/24 (TEST-NET-1)<br>198.51.100.0/24 (TEST-NET-2)<br>203.0.113.0/24 (TEST-NET-3)|[RFC5737]|ドキュメント|
||192.88.99.0/24|[RFC7526]|IPv6 から IPv4 へのリレー（サポート終了）|
||198.18.0.0/15	|[RFC2544]|ベンチマーク テスト|
||240.0.0.0/4	|[RFC5735]<br>/ [RFC1112]|将来の使用（クラス E）|

[RFC1918]: https://tools.ietf.org/html/rfc1918
[RFC6598]: https://tools.ietf.org/html/rfc6598
[RFC6890]: https://tools.ietf.org/html/rfc6890
[RFC5737]: https://tools.ietf.org/html/rfc5737
[RFC7526]: https://tools.ietf.org/html/rfc7526
[RFC2544]: https://tools.ietf.org/html/rfc2544
[RFC5735]: https://tools.ietf.org/html/rfc5735
[RFC1112]: https://tools.ietf.org/html/rfc1112

## 範囲外を設定してしまうと、、、

### 事象1. VPCピアリングによるVPC間通信ができないことがある

2022/04/12時点で、VPCピアリングを双方向で設定するとステータス上は「有効」と表示されるものの、内部で自動的に作成されるはずのルートリソースができておらず、通信が各々のVPCへルーティングできないといった事象が確認されています。

なお、本事象発生時にCloud Consoleから手動でルートの作成を試みたものの、自動作成されるルートと同じ内容のモノをあとからユーザが手動で作成する、というのは叶わずの状況でした。（ネクストホップとしてVPCピアリングリソースを指定することができなかったため）

そのため、この事象に対する有効な対処方法としては、結局のところ「有効なIPアドレス範囲に変更したサブネットへ作り直してください」となってしまうのかなと思います。

## 終わりに

普段から「(RFCは良く知らないけど)プライベートIPアドレスは "おまじない" の意味も込めて10.0.0/8、172.16.0.0/12、192.168.0.0/16の範囲から指定するようにしています」という慎重派の方には無縁の問題ではございますが、中には「プライベートネットワークなんだからIPアドレスは何を指定しても良いでしょ」という方もいるかと思います。

今回ご紹介したとおりGoogle Cloudでは、いくつかのRFCを元に有効なIPアドレス範囲が定義されており、それを逸脱すると諸問題が起きてしまう可能性がありますのでぜひご留意いただければと思います。

---

- Google Cloud は、Google LLC の商標または登録商標です。
- その他、本資料に記述してある会社名、製品名は、各社の登録商品または商標です。

