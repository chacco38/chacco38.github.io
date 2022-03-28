---
title: "AKS on Azure Stack HCIの一般提供(GA)が開始しました"
date: 2021-07-01T00:00:00+09:00
tags: ["Azure", "Azure Stack HCI", "Azure Kubernetes Service(AKS)"]
draft: false
---

今更ですが Microsoft Build 2021 にて AKS(Azure Kubernetes Service) on Azure Stack HCI の一般提供が発表されておりました。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://news.microsoft.com/ja-jp/2021/05/26/210526-microsoft-loves-developers-welcome-to-build-2021/" frameborder="0" scrolling="no"></iframe>

この機能によって Kubernetes のコントロールプレーンは Azure マネージドで提供されつつも、ワークロードはオンプレの Azure Stack HCI 上で動かせるようになります。

そのため、PoC などは Azure 上の AKS でフットワーク軽くやって、でも本番はオンプレの AKS on Azure Stack HCI でカッチリ、っていうシナリオも描けるようになってきた感じでしょうか。Azure Stack HCI が世に出てから2年越し(?)でやっと単なるサーバ製品じゃなく、Azure を冠した意味/メリットがでてきてくれそうですね。