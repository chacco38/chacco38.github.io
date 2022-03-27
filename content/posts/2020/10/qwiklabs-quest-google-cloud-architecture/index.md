---
title: "Qwiklabs クエストの Cloud Architecture: Design, Implement, and Manage を攻略する"
date: 2020-10-30T00:00:00+09:00
tags: ["Google Cloud", "Qwiklabs"]
draft: false
---

Google Cloud Certified Professional Cloud Architect 資格取得に向けた学習の一環で、Google Courses powered by Qwiklabs の「Cloud Architecture: Design, Implement, and Manage」のチャレンジラボクエストに挑戦しました。チャレンジラボの中にはクリア条件が不明瞭で手こずったラボもありましたので同様に詰まっている方の助けになればいいなぁと思います。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://google.qwiklabs.com/quests/124?locale=ja" frameborder="0" scrolling="no"></iframe>

## Qwiklabs クエストを攻略する

次の 7 つのチャレンジラボすべてをクリアするとクエスト攻略となります。

|No.|チャレンジラボ名|
|:---:|---|
|1| [Google Cloud の基本スキル: チャレンジラボ（GSP101）](https://google.qwiklabs.com/focuses/1734?locale=ja&parent=catalog)
|2| [リモート起動スクリプトを使用した Compute インスタンスのデプロイ（GSP301）](https://google.qwiklabs.com/focuses/1735?locale=ja&parent=catalog)
|3| [Deployment Manager を使用したファイアウォールと起動スクリプトの構成（GSP302）](https://google.qwiklabs.com/focuses/1736?locale=ja&parent=catalog)
|4| [Windows の要塞ホストを使用したセキュアな RDP の構成（GSP303）](https://google.qwiklabs.com/focuses/1737?locale=ja&parent=catalog)
|5| [Windows の要塞ホストを使用したセキュアな RDP の構成（GSP303）](https://google.qwiklabs.com/focuses/1738?locale=ja&parent=catalog)
|6| [Kubernetes クラスタでのコンテナ化されたアプリケーションのスケールアウトと更新（GSP305）](https://google.qwiklabs.com/focuses/1739?locale=ja&parent=catalog)
|7| [MySQL データベースの Google Cloud SQL への移行（GSP306）](https://google.qwiklabs.com/focuses/1740?locale=ja&parent=catalog)

### Lab1. Google Cloud の基本スキル: チャレンジラボ（GSP101）

「GSP101」は、Compute Engine の VM インスタンスを作成して、手動でゲスト OS 上に apache2 をインストールするだけのラボです。苦労するところはとくにないかな、と。

<details><summary>解答</summary>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://koejima.com/archives/2307/" frameborder="0" scrolling="no"></iframe>

</details>

### Lab2. リモート起動スクリプトを使用した Compute インスタンスのデプロイ（GSP301）

「GSP301」は、前のラボと同じ内容を起動スクリプトを使って行うラボです。具体的には、Compute Engine の VM インスタンスを作成する際に、startup-script-url オプションにスクリプトを配置した Cloud Storage の URL を指定させて自動で WEB サーバを構築します。

注意点としては、スクリプトを格納した Cloud Storage へアクセスできるように Compute Engine へ権限付与するのを忘れずに、です。

<details><summary>解答</summary>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://koejima.com/archives/2285/" frameborder="0" scrolling="no"></iframe>

</details>

### Lab3. Deployment Manager を使用したファイアウォールと起動スクリプトの構成（GSP302）

「GSP302」は、Deployment Manager を使って Compute Engine の VM インスタンスと Firewall をデプロイするラボです。

筆者は、最後のチェックポイント「Check that Deployment manager includes startup script and firewall resources」がなかなか条件を満たせずに苦労しました。いや、環境自体はデプロイ成功して、外部から WEB アクセスも通っているんですがクリア扱いにならないという…。

結論としては、.jinja ファイルに上記 2 つのリソース定義を含めろよって言っていたようで、Firewall のリソース定義を.yaml から.jinja ファイルへ持っていく必要がありました。（逆に.jinja の VM インスタンス定義を.yaml へ移動でもよかったのかも）

<details><summary>解答</summary>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://koejima.com/archives/2274/" frameborder="0" scrolling="no"></iframe>

</details>

### Lab4. Windows の要塞ホストを使用したセキュアな RDP の構成（GSP303）

「GSP303」は、踏み台サーバ（Bastion）を用意してその踏み台を経由してサーバをデプロイしていくラボです。

文中にリージョンを制限される記載はないものの東日本リージョンではサブネット構築のチェックポイントをクリアできず、米国リージョンに変更する必要がありました。筆者は us-central1 を選択しました。

ちなみに、ラボを進めるにあたり GCP 上の VM インスタンスへの RDP 接続が必要になりますのでプロキシ環境化のクライアントからの実施はやや厳しいかもしれないです。

<details><summary>解答</summary>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://koejima.com/archives/2189/" frameborder="0" scrolling="no"></iframe>

</details>

### Lab5. Kubernetes クラスタへの Docker イメージのビルドとデプロイ（GSP304）

「GSP304」は、Docker イメージのビルド、Container Registory へのイメージプッシュ、Kubernetes クラスタにアプリをデプロイという一覧の操作をしていくラボです。

Container Registory にプッシュする際はイメージ名に`gcr.io/[PROJECT ID]/[IMAGE NAME]:[TAG NAME]`とつける点に注意しましょう。

k8s へのアプリデプロイは全部コマンドを手打ちし終わってから気づきましたが、manifests ディレクトリにサンプル yaml が入っていたのでこちらを編集して`kubectl create -f`してもよかったのかもしれませんね。

<details><summary>解答</summary>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://koejima.com/archives/2210/" frameborder="0" scrolling="no"></iframe>

</details>

### Lab6. Kubernetes クラスタでのコンテナ化されたアプリケーションのスケールアウトと更新（GSP305）

「GSP305」は、k8s へデプロイされているアプリのコンテナイメージ更新およびポッド数の変更操作をしていくラボです。

操作については特筆する点はないかと思います。イメージ変更やレプリカ数の変更が簡単に Cloud Console（GUI）でもできてしまうのは他のメガクラウドと比べて GCP の優秀なところですね。

<details><summary>解答</summary>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://koejima.com/archives/2233/" frameborder="0" scrolling="no"></iframe>

</details>

### Lab7. MySQL データベースの Google Cloud SQL への移行（GSP306）

「GSP306」は、Google Cloud SQL デプロイ、既存 MySQL サーバからのデータ移行、アプリで利用するデータベース切替の一連操作をしていくラボです。

データ移行のところは mysqldump コマンドでデータベースをエクスポートして Cloud Storage にアップロード、Cloud Console（GUI）を使って SQL へデータをインポートしていきましょう。それ以外の操作はチェックポイントの条件を満たしていけば特筆しなくてもクリアできるかと思います。

<details><summary>解答</summary>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://koejima.com/archives/2251/" frameborder="0" scrolling="no"></iframe>

</details>

## 終わりに

やはりクラウドは触って試してみるのがお勉強として一番効率いいと思いますので、ぜひ皆さんも触ってみてください。いやーしかし、GCP もそうなんですがメガクラウドベンダーから提供されている教材は充実していますね。これらが無償で受けられるとは…まいったね、こりゃ。

しかも、Google さんが提供するパートナー認定資格キックスタートプログラムを通じて、今回紹介した「Cloud Architecture: Design, Implement, and Manage」と、その前段のクエスト「Cloud Architecture」の 2 つをクリアすると Professional Cloud Architect 認定資格試験の無料バウチャーまでもらえます。太っ腹だなぁ＾＾；

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://google.qwiklabs.com/quests/124?locale=ja" frameborder="0" scrolling="no"></iframe>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://google.qwiklabs.com/quests/24?locale=ja" frameborder="0" scrolling="no"></iframe>
