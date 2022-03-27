---
title: "GitHub Enterprise Cloud の監査ログ(Audit Log)を CLI を使って取得する"
date: 2021-07-21T00:00:00+09:00
expiryDate: 2021-12-07T00:00:00+09:00
tags: ["GitHub Enterprise Cloud", "監査ログ"]
draft: false
---

今回は GitHub Enterprise Cloud (GHEC) の監査ログ (Audit Log) の取得方法について、です。

監査ログの取得は次のようにいくつか方法はあるのですが、この記事では GitHub さんが公開している GHEC の監査ログを取得するためのコマンドラインインタフェースである GHEC Audit Log CLI を使った方法をご紹介していきたいと思います。

- WEB ポータルからエクスポート
- Audit Log API (GraphQL API または REST API) を実行
- Audit Log CLI を実行 など

https://github.com/github/ghec-audit-log-cli

ちなみに、今回の操作マシンですが AWS CloudShell を使って実行しています。

## さっそく GHEC Audit Log CLI をセットアップしよう

### Step1. 前提パッケージのインストール

手順を見る限り、Node.js が必要っぽいのでインストールします。Linux(RedHat 系)だとこんな感じ。

```text:コマンド実行例（前提パッケージ導入）
$ curl --silent --location https://rpm.nodesource.com/setup_14.x | sudo bash -
$ sudo yum install -y nodejs
```

いや実際は、AWS CloudShell には Node.js がすでに入っているから不要なんですけどね^^;

### Step2. ソースコードの取得とセットアップ

GitHub からソースコードを取ってきて、npm コマンドを使って GHEC Audit Log CLI をインストールしていきましょう。最後の `ghce-audit-log-cli -v` コマンドにてバージョン情報がとってこれれば OK です。

```text:コマンド実行例（ソースコード取得とセットアップ）
$ git clone https://github.com/github/ghec-audit-log-cli.git
$ cd ghec-audit-log-cli
$ sudo npm link
$ ghec-audit-log-cli -v
```

### Step3. パーソナルアクセストークン(PAT)の取得

監査ログを取得したい GitHub Organization の Owner 権限を持つユーザにて、GitHub > Setting > Developer settings > Personal access tokens > Generate new tokens から、指定の通り「read:org」と「admin:enterprise」のスコープを選択した PAT を作ります。

PAT を取得したら設定ファイル `.ghec-audit-log` に情報を書き込みます。`org` には Audit Log を取得したい GitHub Organization 名を、`token` に PAT の値を指定します。念のため補足しますと、GitHub Organization 名は Display Name ではなくて URL にも使われている方です。

```text:コマンド実行例（設定ファイル編集）
$ vi .ghec-audit-log
org: <Your organization's name>
token: <Your token>
```

### Step4. CLI の試し打ち

ここまで来たら、実際にコマンドを実行して監査ログが取得できるか試すだけです。

えいっ！と実行してみると「君のトークンはスコープが足らんぜ」というエラーメッセージが、、、そういえば手順の方にも「The scopes will change depending on what source you are going to use to export the audit logs.」と書かれていましたね^^;

```text:コマンド実行例（CLIの試し打ち）
$ ghec-audit-log-cli
GraphqlError: Your token has not been granted the required scopes to execute this query. The 'organizationBillingEmail' field requires one of the following scopes: ['admin:org'], but your token has only been granted the: ['admin:enterprise', 'read:org'] scopes. Please modify your token's scopes at: https://github.com/settings/tokens.
```

とはいえ、エラーメッセージから何の権限が足りていないのかがわかるため、次のアクションに詰まることはないと思います。(例だと 'admin:org' が足りてないと言われている)

### Step5. CLI が成功するまで Step3~4 を繰り返す

ここではひたすら Step3 と Step4 を繰り返し実行して必要なスコープを追加していきましょう。

ちなみに、毎度毎度いちいち設定ファイルをいじるのも面倒だぜって時は次のように `-o` オプションと `-t` オプションで引数として指定してしまうのもアリかと。ただし、コマンド履歴にトークン情報がのってきちゃうので気を付けること。実行前に履歴残らないように設定しておくとか、ちゃんと後から `history -c` で履歴を消すとかの後始末はしましょう。

```text:コマンド実行例（CLIの試し打ち）
$ ghec-audit-log-cli -o <Your organization name> -t <Your token>
```

ご参考ですが、筆者の環境では PAT に付与した権限は最終的に次のようになりました。ジャジャーン。

![](/images/articles/export-github-enterprise-cloud-audit-log-using-cli/personal-access-token.png)

ということで、無事に GHEC Audit Log CLI を実行できる環境が整いました。

## では改めて GHEC Audit Log を取得しよう

`ghec-audit-log`コマンドを実行して監査ログを取得しましょう。

もちろんリダイレクトさせても良いですが、実行例では `-f` オプションで出力ファイルへの書き込み、`-p` で json を読みやすいフォーマットに変換する(改行やスペースを入れる)オプションを付与してみました。

```
$ ghec-audit-log -f <Output file> -p
```

あとはコレを定期的に実行させるだけですが、、、今回は割愛します^^; ドキュメントでは GitHub Actions を使って定期実行させていますので参考にしても良いかもしれませんね。

## 最後に

ということで、今回は GitHub Enterprise Cloud (GHEC) の監査ログを取得する方法でした。

監査ログを長期(90 日以上)保存したい方はエクスポートが必須となってきますので参考にしてみてはいかがでしょうか。