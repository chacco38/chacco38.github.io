---
title: "プロキシ環境下の Windows 10 マシンから Amazon EKS を CLI で操作する"
date: 2020-05-02T00:00:00+09:00
tags: ["AWS", "Windows", "AWS CLI", "Amazon EKS", "Proxy"]
draft: false
---

みなさん、こんにちは。今回は社内プロキシ環境などから eksctl コマンドや kubectl コマンドを使って Amazon EKS を操作するためのクライアント側の設定方法を記載していきたいと思います。

プロキシ環境下でクラウドベースの開発をしているエンジニアさんはプロキシに泣かされるってこと多いんじゃないかなと思います。いや、ほんとプロキシ関連の調査で時間をとられるのつらいですよね^^; 

## 設定手順

### まずは AWS CLI のセットアップをします

#### 1. AWS CLI をインストールします

[AWS 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/install-cliv2-windows.html)の通りに MSI インストーラをダウンロード＆実行してください。

#### 2. AWS 認証情報を設定します

次のコマンドを実行して認証情報を設定します。

```txt:powershell
PS C:\> aws configure
```

#### 3. AWS CLI のプロキシ設定をします

次のコマンドを実行してから設定を適用するために PowerShell アプリを起動しなおします。

```txt:powershell
PS C:\> setx HTTP_PROXY http://<proxy username>:<proxy password>@<proxy hostname or ip address>:<proxy port>
PS C:\> setx HTTPS_PROXY http://<proxy username>:<proxy password>@<proxy hostname or ip address>:<proxy port>
```

### 次に EKS CLI ツールのセットアップをします

次にパッケージマネージャの Chocolatey を使って eksctl や kubectl などをインストールしていきましょう。

#### 1. Chocolatey をインストールします

公式サイトから[インストールスクリプト](https://chocolatey.org/install.ps1)をダウンロードし、PowerShell を管理者として起動して環境変数を設定した上でインストールスクリプトを実行します。

```txt:powershell(管理者)
PS C:\> $env:chocolateyProxyLocation = "<proxy hostname or ip address>:<proxy port>"
PS C:\> $env:chocolateyProxyUser = "<proxy username>"
PS C:\> $env:chocolateyProxyPassword = "<proxy password>"
PS C:\> Set-ExecutionPolicy RemoteSigned
PS C:\> .\$downloadPath\install.ps1
```

#### 2. Chocolatey のプロキシ設定をします

次のコマンドを実行してプロキシ設定をします。

```txt:powershell
PS C:\> chocolatey config set proxy "<proxy hostname or ip address>:<proxy port>"
PS C:\> chocolatey config set proxyUser "<proxy username>"
PS C:\> chocolatey config set proxyPassword "<proxy password>"
```

#### 3. eksctl などをインストールします

次のコマンドを実行して eksctl などをインストールしていきましょう。

```txt:powershell
PS C:\> chocolatey install -y eksctl aws-iam-authenticator kubernetes-cli
```

ということでここまででひとまずセットアップは終了です。

### では既存の EKS クラスタへの接続確認をしましょう

#### 1. まずは kubeconfig を作成します

次のコマンドを実行して kubeconfig（kubectl 接続設定ファイル）を作成します。

```txt:powershell
PS C:\> aws eks --region $region update-kubeconfig --name $cluster
```

{{< alert >}}
**$region**、**$cluster**部分は自分の環境に沿った値へ変換して実行してください。
{{< /alert >}}

#### 2. kubectl を実行してみよう

次のコマンドなどを実行して情報が取得できたら終わりです。

```txt:powershell
PS C:\> kubectl get svc
PS C:\> kubectl get pods
```

{{< alert >}}
「**error: You must be logged in to the server (Unauthorized)**」で kubectl がエラーになった場合は RABC ではじかれている可能性が高いです。スイッチロールするなりしてアクセスできる状態にした上で再実行してください。
{{< /alert >}}

## 終わりに

ということで、今回は Amazon EKS をプロキシ環境下の Windows 10 マシンから CLI 操作する方法でした。プロキシ環境で戦うエンジニアさんの力に少しでもなれれば幸いです。
