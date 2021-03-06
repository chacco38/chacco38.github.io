---
title: "GKE クラスタ作成時のオプションで Anthos Service Mesh を有効化できるようになりました"
date: 2021-12-16T00:00:00+09:00
lastmod: 2021-12-22T00:00:00+09:00
tags: ["Google Cloud", "Anthos Service Mesh", "Google Kubernetes Engine(GKE)", "Kubernetes", "Istio" ]
draft: false
externalUrl: https://qiita.com/chacco38/items/e33a9c2dced7b6d420ec
---

みなさん、こんにちは。今回はタイトルに掲げたとおり、最近の更新で Google Cloud コンソール(GUI)から Google Kubernetes Engine(GKE) Standard クラスタを作成する際に、次のようなすてきなオプションがプレビュー機能として追加されました。今回はこのオプションを使ってどのような構成が作られるのか実験したので共有したいと思います。

![01-anthos-service-mesh-enable-option.png](images/01-anthos-service-mesh-enable-option.png)

## いきなりですが、結論です！

### Q1. In-cluster とマネージドコントロールプレーンのどちらで構築される？

答え、マネージドコントロールプレーンにて構築されます。現時点でこの設定を変更することはできません。

### Q2. Anthos Service Mesh のバージョンは？

答え、GKE でリリースチャンネルを採用した場合は GKE と同じチャンネルになります。GKE で静的リリースを採用した場合は Reguler チャンネルとなります。現時点でこの設定を変更することはできません。

### Q3. カスタム CA を扱うことはできるか？

答え、扱えません。マネージドコントロールプレーンでの導入となるため、カスタム CA を扱うことができる Istio CA を選択することはできません。

### Q4. 限定クラスタにした場合は別途 15017/TCP を許可する必要があるか？

答え、不要です。ルールを追加しなくてもサイドカー自動インジェクションは問題なく動きます。

### Q5. Ingress ゲートウェイはデフォルトで作られるか？

答え、Ingress ゲートウェイはデフォルトでは作られません。別途ユーザにてデプロイする必要があります。

## 終わりに

今回は GKE クラスタ作成時の Anthos Service Mesh 有効化オプションの実験結果の共有でしたがいかがだったでしょうか。

GKE と Anthos Service Mesh で採用するリリースチャンネルを変えたい、カスタム CA を使いたいといったケースでは従来どおり CLI を利用する必要がありますが、これらの要件がなければ GUI からポチポチするだけでとっても簡単に環境を作れるようになりそうですね。

{{< alert >}}
2021 年 12 月時点ではGUI の Anthos Service Mesh 有効化オプションはプレビュー段階であり、今回紹介した挙動から変わる可能性がありますのでご注意ください。
{{< /alert >}}

---

- Google Cloud は、Google LLC の商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
