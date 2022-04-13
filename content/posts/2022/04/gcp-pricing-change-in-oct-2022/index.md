---
title: "2022年10月1日に施行されるGoogle Cloudの価格改定について"
date: 2022-04-13T00:00:00+09:00
lastmod: null
tags: ["Google Cloud"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回は2022年3月14日に発表されたGoogle Cloudの価格改定についてのお話です。一部のニュースメディアでは「**大幅値上げ**」と取り上げられていますが、実際のところ「**日本での利用にあたって**」どの程度影響がありそうかを見やすくしてみましたのでご紹介していきたいと思います。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://cloud.google.com/blog/products/infrastructure/updates-to-google-clouds-infrastructure-pricing" frameborder="0" scrolling="no"></iframe>

## Cloud Storage(GCS)の価格改定

|No.|項目|値上げ/値下げ|
|:---:|:---|:---:|
|1|下り（外向き）の通信に対する無料枠の拡大(1GB ⇒ 100GB)|**<font color=blue>無料化:arrow_down:**<br>(MAX$0.23/GB	 ⇒ $0)|
|2|マルチリージョンのNearline Storage料金の改定|**<font color=red>50%UP:arrow_up:**<br>($0.01/GB ⇒ $0.015)|
|3|マルチリージョン(asia)のColdline Storage料金の改定|**<font color=red>25%UP:arrow_up:**<br>($0.007/GB ⇒ $0.00875)|
|5|マルチリージョン(asia)のArchive Storage料金の改定|**<font color=blue>25%DOWN:arrow_down:**<br>($0.004/GB ⇒ $0.003)|
|7|デュアルリージョン(asia1)のStandard Storage料金の改定|**<font color=red>10%UP:arrow_up:**<br>($0.046/GB ⇒ $0.0506)|
|9|デュアルリージョン(asia1)のNearline Storage料金の改定|**<font color=red>10%UP:arrow_up:**<br>($0.032/GB ⇒ $0.0352)|
|11|デュアルリージョン(asia1)のColdline Storage料金の改定|**<font color=red>10%UP:arrow_up:**<br>($0.012/GB ⇒ $0.0132)|
|13|デュアルリージョン(asia1)のArchive Storage料金の改定|**<font color=blue>13%DOWN:arrow_down:**<br>($0.0065/GB ⇒ $0.0056)|
|14|Coldline StorageクラスBオペレーション料金の改定|**<font color=red>100%UP:arrow_up:**<br>($0.05 ⇒ $0.1)|
|15|Coldline StorageクラスAオペレーション料金の改定<br>(シングルリージョンの場合)|**<font color=red>100%UP:arrow_up:**<br>($0.1 ⇒ $0.2)|
|16|Coldline StorageクラスAオペレーション料金の改定<br>(マルチリージョン/デュアルリージョンの場合)|**<font color=red>300%UP:arrow_up:**<br>($0.1 ⇒ $0.4)|
|17|上記以外のクラスAオペレーション料金の改定<br>(マルチリージョン/デュアルリージョンの場合)|**<font color=red>100%UP:arrow_up:**<br>($0.05 ⇒ $0.1)|
|17|asia/asia1ロケーションのデータレプリケーション料金の有料化|**<font color=red>有料化:arrow_up:**<br>($0 ⇒ $0.08/GB)|

## Cloud Load Balancingの価格改定

|No.|項目|値上げ/値下げ|
|:---:|:---|:---:|
|1|ロードバランサで処理された下り（外向き）データの有料化|**<font color=red>有料化:arrow_up:**<br>($0 ⇒ MAX$0.012/GB)|**<font color=red>有料化:arrow_up:**<br>($0 ⇒ $0.0011/h)|

## Network Intelligence Centerの価格改定

|No.|項目|値上げ/値下げ|
|:---:|:---|:---:|
|1|ネットワークトポロジおよびパフォーマンスダッシュボード機能の有料化|**<font color=red>有料化:arrow_up:**<br>($0 ⇒ $0.0011/h)|

## Compute Engine(GCE)の価格改定

|No.|項目|値上げ/値下げ|
|:---:|:---|:---:|
|1|永続ディスクのリージョンスナップショット料金の改定|**<font color=red>92%UP:arrow_up:**<br>($0.026/GB ⇒ $0.05/GB)|
|2|　〃　のマルチリージョンスナップショット料金の改定|**<font color=red>150%UP:arrow_up:**<br>($0.026/GB ⇒ $0.065/GB)|
|3|　〃　のリージョンアーカイブスナップショット機能の新設|**<font color=blue>26%DOWN:arrow_down:**<br>($0.026/GB ⇒ $0.019/GB)|
|4|　〃　のマルチリージョンアーカイブスナップショット機能の新設|**<font color=blue>7%DOWN:arrow_down:**<br>($0.026/GB ⇒ $0.024/GB)|

## 終わりに

今回は他のメガクラウドとの料金比較まではしていないため、市場に対する割高感があるかまでは何とも言えないものの、既存のGoogle Cloudユーザから見るとたしかにメディアで語られているとおり大幅な値上げと言われても仕方ないように感じますね^^;

とはいえ、ほんの一部ではありますがArchive Storageやアーカイブスナップショットといった既存よりも安くなってくる部分もあるため、前向きにこれを良い機会だととらえてデータのライフサイクルを見直し、これらの長期保管用サービスの活用をもっと推進していけば今よりコストをおさえることも可能なのではなでしょうか。

以上、2022年10月1日に施行されるGoogle Cloudの価格改定についてのお話でした。

---

- Google Cloud は、Google LLC の商標または登録商標です。
- その他、本資料に記述してある会社名、製品名は、各社の登録商品または商標です。


