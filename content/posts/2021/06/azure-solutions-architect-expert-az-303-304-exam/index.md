---
title: "Microsoft 認定 Azure Solutions Architect Expert (AZ-303/AZ-304) 受験メモ"
date: 2021-06-05T00:00:00+09:00
tags: ["Azure", "資格"]
draft: false
---

あれは Azure を業務で扱いだして 2~3 か月くらい経った頃、そろそろ Solutions Architect Expert を取りに行かねばな、と思い立って AZ-303 (2021/1/5) と AZ-304 (2020/11/25) の試験を受けに行った時のメモです。

## 受験メモ

### 受験者情報

- AWS 資格 5 種 (CLF、SAA、SAP、DOP、SCS)
- Azure 資格 3 種 (AZ-900、AZ-104、AZ-400)
- GCP 資格 1 種 (PCA)

### 結果

- AZ-303: 760 点 (合格)
- AZ-304: 895 点 (合格)

### 試験対策例

1. 合格者のブログを漁って勉強の計画などを立てる (サボったので 0 時間)
1. 試験のアウトラインを読んで不安な箇所がないかを確認する (約 1 時間)
1. 自信のない部分を Microsoft Learn などでお勉強する (サボったので 0 時間)
1. Udemy などで評価の高い問題集をやる (サボったので 0 時間)

### 感想

- AZ-303 は、AZ-304 よりも偏りなくまんべんなくいろんな分野の問題が出た印象。テクノロジーと銘打つ試験の通り、AZ-304 よりも個々のサービスについて深くて細かいノウハウを要求された印象。
- AZ-304 は幅広く、いろいろな分野の問題が出てくるが、Azure AD のハイブリッドクラウドの設計が感覚的に 1 割くらいと多かった印象。デザインと銘打つ試験の通り、細かい仕様よりもシステムを設計する上での考え方などが重要な印象。上流設計をメインでされている方は AZ-303 よりも余裕をもって臨めるかと。
- Azure は AZ-303/AZ-304 の 2 つの試験をパスしないと SA になれないので正直 AWS や GCP よりも手間がかかりましたね。試験の難易度は感覚的に AWS >>> Azure = GCP の印象。※あくまで個人的な意見なのであしからず。

### アドバイスなど

- AZ-303 と AZ-304 の出題範囲はおおよそ同じなので、2 つの試験はあまり間隔を開けずに受験することをオススメします。
- AZ-303 は、たとえば、SQL サーバを冗長化するには RG は同じである必要があるか？SQL サーバをリージョン間で冗長化するための前提条件は？みたいな感じの細かい知識を求める問題が多いです。知らないと勘になってしまう(=選択肢は絞れるけど、どんなに考えても導き出せない)ので試験ガイダンスにしたがって穴を埋めるのが良いかと思います。
- AZ-304 は、FgCF (Financial-grade Cloud Fundamentals) や Azure Well-Architecture フレームワークの知識を叩き込んでいけばどうにかなると思います。また、他クラウドのべスプラが頭に入っている方であれば、Azure についてそのままズバリを知らなくてもこうあるべきだ、を考えれば答えが導き出せるかも！？

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://github.com/nakamacchi/fgcf" frameborder="0" scrolling="no"></iframe>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://docs.microsoft.com/ja-jp/azure/architecture/framework/" frameborder="0" scrolling="no"></iframe>

- 本試験を受けた知人が以下の問題集から似たような問題がちょいちょい出てきたので点数の底上げに役立った、とのこと。評価も高いですし勉強素材の選択肢の1つとして検討してみても良いかも！？

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://www.udemy.com/course/az-303-microsoft-azure-architect-technologies-practice-test/" frameborder="0" scrolling="no"></iframe>

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" src="https://hatenablog-parts.com/embed?url=https://www.udemy.com/course/az-304-azure-architect-design-practice-tests/" frameborder="0" scrolling="no"></iframe>

## 終わりに

繰り返しになりますが、Microsoft 赤間氏が GitHub で公開してくれている FgCF (Financial-grade Cloud Fundamentals) のコンテンツは Azure を設計する上で重要なエッセンスがギュッと詰まっていて、金融って銘打ってますけど他業界でも十分使える内容となっています。試験対策とか置いといて、Azure って何から勉強すればいいの？という方には「まずは FgCF を見んしゃい！」と言わせていただきたいですね。

ということで、Microsoft 認定 Solutions Architect Expert (AZ-303/AZ-304) を受験したときのお話でした。

、、、余談ですが、テクノロジー (AZ-303) の方はノーガード戦法だと辛いだろうなと思って試験前に大型連休を挟んだ　はず　だったんですけどね、、、いやー、合格できてラッキーでした^^;