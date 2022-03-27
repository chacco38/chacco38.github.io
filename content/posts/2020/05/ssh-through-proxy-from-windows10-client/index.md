---
title: "プロキシ環境下の Windows 10 マシンから外部へ SSH でアクセスする"
date: 2020-05-01T00:00:00+09:00
tags: ["Windows", "TeraTerm", "SSH", "Proxy"]
draft: false
---

みなさん、こんにちは。今回は社内プロキシ環境下から Amazon EC2 などの外部環境へ SSH でアクセスしたいといった場合の接続方法です。なお、今回は TeraTerm を利用した手順となっています。

## 接続手順

**1. TeraTerm をインストールする際に「TTProxy」にチェックして入れます**

![](images/select-component.png)

**2. TeraTerm を起動して「設定 > プロキシ」を選択します**

![](images/config-menu.jpg)

**3. プロキシ情報を入力します**

![](images/proxy-setup.png)

**4. あとは「ファイル > 新しい接続」からいつも通りアクセスすれば OK です**

![](images/new-connection.png)

## 終わりに

今更感はありますが「プロキシ環境下の Windows 10 マシンから外部へ SSH でアクセスする方法」でした。ちなみに今のご時世、EC2 などへのアクセスであれば SSH ポートは開けずに System Manager セッションマネージャなどを使う、が正解だと思いますけどね。