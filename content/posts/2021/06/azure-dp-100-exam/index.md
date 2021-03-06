---
title: "Microsoft Certified Azure Data Scientist Associate (DP-100) 受験メモ"
date: 2021-06-05T00:00:00+09:00
tags: ["Azure", "資格"]
draft: false
---

Microsoft Certified Azure Data Scientist Associate (DP-100) を 2021/5/28 に受験していたのでその時のメモです。ほんとは違う試験を受けに行く予定だったので情報少ないです。すんません。

## 受験メモ

### 受験者情報

AWS、Azure、Google Cloudの三大クラウド上級資格持ち、とくにAWS資格は全種コンプしているので一見するとけっこう良さげな(クラウド歴1年の)新米クラウドエンジニアです。

|No.|項目|取得数|取得済み資格/合格済み試験|
|:---:|:---:|:---:|:---:|
|1|AWS資格|12|CLF、SAA、SOA、DVA、SAP、DOP、<br>SCS、ANS、AXS、DBS、DAS、MLS|
|2|Azure資格|4|AZ-900、AZ-104、AZ-303/304、AZ-400|
|3|Google Cloud資格|3|ACE、PCA、PDE|

### 結果

- 781 点 (合格)

### 試験対策例

1. 合格者のブログを漁って勉強の計画などを立てる (うっかりで 0 時間)
1. 試験のアウトラインを読んで不安な箇所がないかを確認する (うっかりで 0 時間)
1. 自信のない部分を Microsoft Learn などでお勉強する (うっかりで 0 時間)
1. Udemy などで評価の高い問題集をやる (うっかりで 0 時間)

### 感想

- Azure Machine Learning の知識が無くても答えられる設問が多かった印象でした。たとえば、Python のコードが出てきて実装コードの穴あきを埋めよとか、トレーニングパイプライン/推論パイプラインの穴あきを埋めよとか、xx のケースだとモデルデプロイ先のアーキテクチャとして最適なものはどれか？みたいな。いやほんと、Azure 以外でも通用するような一般知識を問われるケースが多かったです。

### アドバイスなど

- DP-100 は、Azure Machine Learning に特化した試験なのではっきり言って範囲は広くないです。ワークスペースをどうやってセットアップするか、セットアップ後はどうやってモデルのトレーニングを始めるのか、モデルの精度がイマイチだった場合にどういう手段が取れるのか、トレーニングが終わったモデルを実際にアプリから呼び出して使う際はどうするか、とかいった一連の流れを問われます。ここら辺の分野がまったくの素人ですって方は、時間効率は悪いですが MS Learn でもひととおり学べるので目を通しておくと良いでしょう。
- もし仮に勉強する時間が取れていたら、Udemy で評価の高い英語の問題集を購入してやっていたかと思います。Solutions Architect とかの人気試験ならまだしも、この手の試験は日本語で質の良い問題集はまずないと思った方がいいですし。

## 最後に

**試験を申し込むときは似たような名前の試験があるので最新の注意を払いましょう！**<br>
**試験を申し込むときは似たような名前の試験があるので最新の注意を払いましょう！**

重要なことなので 2 回書きました。本当は私も Power BI の試験 (D **A** -100) を申し込んだつもりだったんですよね、、、試験を開始して 5 問くらいやったあたりで「はわわわ、こいつはおかしい！Azure Machine Learning に関する問題しか出てこないんですけどーーーー！」と内心めちゃくちゃ焦ってました。でも、なんとか合格できたのでラッキーでしたね^^;

ということで、Microsoft Certified Azure Data Scientist Associate (DP-100) を受験したときのお話でした。