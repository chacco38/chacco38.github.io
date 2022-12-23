---
title: "Google Cloud Shellにてlessコマンドからviエディタを起動する方法"
date: 2022-04-06T00:00:00+09:00
lastmod: 2022-12-17T00:00:00+09:00
tags: ["Google Cloud", "Cloud Shell"]
draft: false
externalUrl: https://qiita.com/chacco38/items/3b1a2922d826e3ad52cd
---

みなさん、こんにちは。Google CloudのCloud Shellでは、lessコマンドでファイル参照中に内容を編集しようと `v` キーを押してエディタを起動するとemacsが起動してきます。もちろんemacsも良いエディタですが、Red Hat系のLinuxに慣れている方からすると「lessで `v` を押したらviが起動してきてほしい！てか、moreはviが起動するのになんでlessはviじゃないんだよ！」と思う方もきっといるはずです。今回はそんなときの解決方法を紹介していきたいと思います。

## エディタを切り替える方法

それでは解決方法ですが、環境変数 `VISUAL` に起動したいコマンド(今回はvi)を指定してください。これでlessからviエディタを起動させることができるようになります。

**実行例）**

```bash
export VISUAL="vi"
```

## 起動時に自動で設定する方法

とはいえ、Cloud Shellを起動するたびに毎回環境変数の設定をするのは面倒です。そんなときは設定ファイル `~/.bashrc` の末尾に次の1行を追記しましょう。これでCloud Shellが起動した際に自動で適用されるようになります。めでたしめでたし。

**作成例）~/.bashrc**

```txt
export VISUAL="vi"
```

## 終わりに

いまさらの情報でしたがいかがだったでしょうか。lessがなんか使いにくいんだよなとお困りの方は参考にしていただければと思います。以上、Google Cloud Shellにてlessコマンドからviエディタを起動する方法でした。

---

- Google Cloud は、Google LLC の商標または登録商標です。
- その他、本資料に記述してある会社名、製品名は、各社の登録商品または商標です。
