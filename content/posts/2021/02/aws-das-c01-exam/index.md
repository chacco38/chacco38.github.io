---
title: "AWS Certified Data Analytics - Specialty (DAS-C01) 受験メモ"
date: 2021-02-20T00:00:00+09:00
lastmod: null
tags: ["AWS", "資格"]
draft: false
externalUrl: null
---

AWS資格コンプを目指して AWS Certified Data Analytics - Specialty (DAS) を 2021/2/16 に受験してきました。その時のメモです。

## 受験メモ

### 受験者情報

AWS、Azure、Google Cloudの三大クラウド上級資格を持っているので一見ではそこそこのエンジニアに見えるものの、実際はAWSを業務で扱いだして約10か月、Azureは約5か月、Google Cloudは約3か月の駆け出しクラウドエンジニアです。

|No.|項目|取得数|取得済み資格/合格済み試験|
|:---:|:---:|:---:|:---:|
|1|AWS資格|9|CLF、SAA、SOA、SAP、DOP、SCS、ANS、AXS、DBS|
|2|Azure資格|4|AZ-900、AZ-104、AZ-303/304、AZ-400|
|3|Google Cloud資格|1|PCA|

### 結果

765点 (合格)

### 試験対策例

|No.|対策項目|実施時間(例)|
|:---:|:---|:---:|
|1|公式のサンプル問題をやる|約0.5時間|
|2|問題集をやる|約2時間 x 5日|
|3|不安な部分をサービス別資料を読んで補強する|約2時間|

### アドバイスなど

データ分析の通常プロセス（収集、格納、加工、分析、可視化）にかかわる問題がまんべんなく出てきた感じです。Kinesis(Data Stream、Data Firehose、Data Analytics)、S3、Redshift、EMR、Glue、Athena、QuickSight、ElasticSearchとKibanaあたりの知識は必須です。また、EMRなどで使われているApache系ツールについてもざっくり何やるものかくらいは把握しておきましょう。

ちなみに、点数取れてないので偉そうなことは言えませんが、ポイントとしては以下のような感じかと思います。

- 収集はストリーミングで取り込むならKinesis系、たまにMSK(Kinesisで扱えない大きなサイズとか)、一発ガツンと移行ならSnowballなどのデータ移行ツール。
- 加工はEMR(Hadoop、Spark、Hive、Presto等のApache系ツール)が基本、楽したいときはGlue、たまにBatchで短時間回すといったケースもアリ。EMRのワーカーやBatchは、コスト面からスポットの利用や不要なら止めておく。Athenaとかで分析しやすいように取り込み段階で形式変更、結合、圧縮を。Redshiftへ取り込みやすいようにスライス数の倍数に分割とかも。
- 格納先はS3、Redshift(複雑なSQL使う系)、DynamoDB(細かいデータたくさん系)
- 可視化はQuickSight、ESならKibana、場合によりJupyterノートブック的なのも。

## 終わりに

点数取れてないので偉そうなことは言えませんが、しっかり対策していけばそこまで難しい試験ではない印象でした。AWS資格コンプまであと2個、やっと終わりが見えてきました。来週、いっきに2試験を受けてコンプしてきます。
