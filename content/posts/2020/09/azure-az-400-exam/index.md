---
title: "Microsoft Certified: DevOps Engineer Expert (AZ-400) 受験メモ"
date: 2020-09-27T00:00:00+09:00
lastmod: null
tags: ["Azure", "資格"]
draft: false
externalUrl: null
---

Azure を業務で扱い出して約2週間が経ち、そろそろ上級資格を狙ってみようと Microsoft Certified: DevOps Engineer Expert (AZ-400) を 2020/9/25 に受験してきたときのメモです。

## 受験メモ

### 受験者情報

AWSを業務で扱いだして半年弱、Azureは約2週間、AWS認定資格的には対外的にそれっぽく見えてきたクラウドエンジニアです。

|No.|項目|取得数|取得済み資格/合格済み試験|
|:---:|:---:|:---:|:---:|
|1|AWS認定資格|4|SAA、SAP、DOP、SCS|
|2|Azure認定資格|2|AZ-900、AZ-104|

### 結果

777点 (合格)

### 試験対策例

|No.|対策項目|実施時間(例)|
|:---:|:---|:---:|
|1|Microsoft Learnで勉強する|約2時間 × 3日間|

### アドバイスなど

App InsightsやAMonitorを使ったアプリのトラシュー関連の問題をよく見た一方で、Azure DevOpsサービスに関する問いは少なかった印象です。一般的なCI/CDツールに関する問いも散見されました。デプロイ方式（B/G、Canaryなど）は概要さえ押さえておけばOKだったように思います。

DevOpsの試験だから、おそらくAzure DevOpsだけ勉強していけば大丈夫だろうと油断していると足元を救われると思います。基本中の基本ですが、憶測だけで適当な範囲を勉強していくのではなく、認定資格の画面からダウンロードできる「認定資格スキルのアウトライン」から出題範囲や配点割合をちゃんと確認してから勉強をすることをオススメします。

なお、MS Learn教材をひととおりやればある程度の知識は身に付くと思いますが、一般的にCI/CDで使われるツール（Git、Subversion、Maven、SonarQube、Jenkins、jMeter、Selenium、Terraform、Ansibleなど）の概要くらいは頭に入れておきましょう。AWSと違ってAzure固有のサービスに関する知識ではなく一般知識を問われる印象です。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://docs.microsoft.com/ja-jp/learn/certifications/devops-engineer/" frameborder="0" scrolling="no"></iframe>

## 終わりに

本当はAzure Solution Architect Expertの方を取るべき立場なのですが正直試験を2つも受けるのがやや面倒だったということもあり、手っ取り早く上級資格保有者になるためDevOps Engineer Expertを選択しました^^;

いやー、DevOpsというくらいなのでおそらくAzure DevOpsサービスについての設問ばかりであろうと勝手な憶測の元、Azure DevOpsの勉強のみをして本試験に挑んでしまったのは完全に失敗でしたね。

フタを開けてみればAzure DevOpsに関する出題割合は1割くらいしかなく、変な汗がいっぱい出て完全に負け戦の模様だったのですが、脳内にある知識をフル活用してなんとか合格をもぎ取ることができてとてもラッキーでした。

とはいえ、このままではいずれ失敗をするので次回からは試験の出題範囲や配点割合をしっかり目を通してから勉強をすすめようと思う今日この頃でした^^; めでたし、めでたし。