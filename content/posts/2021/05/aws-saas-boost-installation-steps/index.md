---
title: "AWS SaaS Boost がオープンソースとして公開されたのでさっそくいじってみよう"
date: 2021-05-29
tags: ["AWS", "AWS SaaS Boost"]
draft: false
---

みなさん、こんにちは。ちょっと前に AWS さんのブログを漁ってたら「AWS SaaS Boost がオープンソースとしてリリースされました」っていう記事が目に入ってきたんですよね。

https://aws.amazon.com/jp/blogs/news/aws-saas-boost-released-as-open-source/

普通は AWS SaaS Boost をサービスとして利用すればいいだけですし、へー、コードが公開されたんだ、でスルーしようとも思ったんですが、なんとなく気になったんです。もしかするとリポジトリをフォークして魔改造して使うときが来るかもしれない。いや、来ないだろうけど、、、

ということで、そんな時に備えるためにちゃちゃっと改造して、改造したものを動かしてみるところまで試せたらいいなーと思って筆をとってみた次第です。

## そもそも "AWS SaaS Boost" って何者よ？

AWS SaaS Boost というのは、自分の持っているアプリケーションを SaaS 化したいなー！というときに強い味方になってくれるサービスです。

https://aws.amazon.com/jp/partners/saas-boost/

AWS さんのブログに書いてある内容をそのまま載せちゃいますが、要は、こちらのサービスが **SaaS 化する際に作りこまないといけない機能を補完してくれる** ってことなんですね。

> 既存のアプリケーションを SaaS Boost にインストールするだけで、テナント管理、デプロイ、テナントごとの分析、ビリング（請求）、メータリングがすべてセットアップされ、すぐに使用できるようになります。これにより、ソリューションを再構築するコストをかけることなく、より迅速に、製品を SaaS モデルで市場に提供することが可能になります。

https://aws.amazon.com/jp/blogs/news/transforming-your-monolith-to-saas-with-aws-saas-boost/

ということでユーザは、**よりアプリのコア機能開発に時間を割くことができるようになる** ってことなので、使うメリットは十分にあるんじゃなかろうかと思います。

## まずはオリジナルのまま環境を作ってみる

ん？改造するんじゃないの？と思った方は正解！でも、改造する前に絶対やっておくべきプロセスです。特に若いオープンソースにあるあるだと思いますけど、「手順通りにやったのに動かないぜーー！」ということがあります。ということで、改造云々の前に導入するにあたって追加の前提条件や手順がないのかを確認しておきたいと思います。

ということで、こちらのドキュメントに基本沿って作っていきたいと思います。ちなみに、この記事の Step とドキュメント上の Step の数字は一致するように書いてます。題名は若干違いますが、、、

https://github.com/awslabs/aws-saas-boost/blob/main/docs/getting-started.md

### Step0: 環境の準備

今回は Amazon Linux 2 の EC2 インスタンスを用意しました。メモリは 4GB 以上必要なようです。

### Step1: 前提パッケージの導入

公式ドキュメントにリスト掲載されているパッケージをインストールしていきましょう。

> - Java 11 Amazon Corretto 11
> - Apache Maven
> - AWS Command Line Interface version 2
> - Git
> - Node 14.15 (LTS)
> - Yarn

公式ドキュメントにはコマンドラインまでは書かれてませんけど Amazon Linux 2 だと下の例のような感じです。補足ですが、筆者の環境だと OpenJDK 8 がデフォルトの java のバージョンになってたので、必要に応じてコマンド例のように java バージョンを切り替えて使ってください。永続化はしてないので注意。

**実行例）Amazon Linux 2 の場合**

```text:コマンド実行例（AmazonLinux2）
$ sudo yum update -y
$ sudo yum install -y java-11-amazon-corretto maven
$ sudo alternatives --config java
$ export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto.x86_64/

$ curl --silent --location https://rpm.nodesource.com/setup_14.x | sudo bash -
$ sudo yum install -y nodejs

$ curl --silent --location https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
$ sudo yum install -y yarn
```

AWS CLI v2 は最初から入っている環境であったため手順を省いてしまいましたが、もし自分で導入される際はこちらをご参照くださいね。

https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/install-cliv2-linux.html

### Step2: ソースコードを取得

AWS SaaS Boost のソースコードを GitHub からダウンロードします。
ここは公式ドキュメントにコマンドラインが書かれているのでそのまま実行すれば OK なんですが、今回はオリジナルからフォークしてきた個人のリポジトリを使うようにしましたよ、と。

**実行例）Amazon Linux 2 の場合**

```text:コマンド実行例（AmazonLinux2）
$ git clone https://github.com/chacco38/aws-saas-boost.git ./aws-saas-boost
```

### Step3: アプリをビルド＆デプロイ

AWS CLI の設定をした上で対話型インストーラを用いて AWS SaaS Boost を導入していきます、、、って、このタイミングでシステム要件出してくるのか、まぁいいけどｗ

> The system where you run the installation should have at least 4 GB of memory.

気を取り直して、コマンドを実行していきましょう。こんな感じです。ちなみに、出力形式を「YAML」にすると install.sh がエラーになったので「JSON」に設定してます。

**実行例）Amazon Linux 2 の場合**

```text:コマンド実行例（AmazonLinux2）
$ aws configure
$ aws configure set output json

$ cd aws-saas-boost
$ sh -x install.sh
```

install.sh スクリプトの処理が進むとインストーラ起動オプションの入力を迫られます。今回は「1」を選択。

**出力例）インストーラ起動オプション入力時**

```text:スクリプト出力例（インストーラ起動オプション入力時）
===========================================================
Welcome to the AWS SaaS Boost Installer
Setting version to v0 as it is missing from the git properties file.
Installer Version: d64277c-dirty, Commit time: 2021-05-19T21:15:02+0000
Checking maven, yarn and AWS CLI...
Environment Checks for maven, yarn, and AWS CLI PASSED.
===========================================================
1. New AWS SaaS Boost install.
2. Install Metrics and Analytics in to existing AWS SaaS Boost deployment.
3. Update Web Application for existing AWS SaaS Boost deployment.
4. Update existing AWS SaaS Boost deployment.
5. Delete existing AWS SaaS Boost deployment.
6. Exit installer.
Please select an option to continue (1-6): 1
```

次に各パラメータの入力を求められるので適当に設定していき、、、

**出力例）パラメータ入力時**

```text:スクリプト出力例（パラメータ入力時）
Directory path of saas-boost download (Press Enter for '/home/ssm-user/aws-saas-boost') : *****
Enter name of the AWS SaaS Boost environment to deploy (Ex. dev, test, uat, prod, etc.): *****
Enter the email address for your AWS SaaS Boost administrator: *****
Enter the email address address again to confirm: *****
Would you like to setup a domain in Route 53 as a Hosted Zone for the AWS SaaS Boost environment (y or n)? *****
Would you like to install the metrics and analytics module of AWS SaaS Boost (y or n)? *****

If your application requires a FSX for Windows Filesystem, an Active Directory is required.
Would you like to provision a Managed Active Directory to use with FSX for Windows Filesystem (y or n)? *****
```

最後にパラメータの確認をしてから「continue」を意味する「y」を入力したら、あとは何もせずに完了を待つだけです。筆者の環境だと完了まで 15 分弱かかりました。

**出力例）入力パラメータ確認時**

```text:スクリプト出力例（入力パラメータ確認時）
Would you like to continue the installation with the following options?
AWS SaaS Boost Environment Name: *****
Admin Email Address: *****
Route 53 Domain for AWS SaaS Boost environment: *****
Install Metrics and Analytics: *****
Amazon Quicksight user for setup of Metrics and Analytics: *****
Setup Active Directory for FSX for Windows: *****
Enter y to continue or n to cancel: y
```

install.sh スクリプトが完了したらこちらの EC2 インスタンス上での操作は一旦終了のようです。ということで、管理者 Email アドレスに新しいメールが届いていると思うのでメールボックスを見に行きましょう。

### Step4: AWS SaaS Boost への初回ログイン

ここからは Web ブラウザを中心に作業を行っていきます。本ステップでは管理者へ届いたメール本文に書かれている URL へアクセスする、ログインする、そしてパスワード変える、それだけです。入れたことを確認したら終わりです。

公式ドキュメントでは Step5 以降で AWS SaaS Boost アプリ上の設定をしていくわけですが、今回は導入までの流れを確認したかっただけなので以降の手順は割愛します。

## 余談、導入成功までの過程で出たエラー

ということで、今回導入するにあたって出てきたエラー 2 件のご紹介をしたいと思います。

### Case1: 「Error with build of Java installer for SaaS Boost」というエラーメッセージを出力してスクリプト停止

Step1 で必要なパッケージをインストールだけしてスクリプトを実行したらこんな感じでエラーになったんです。mvn コマンドがエラーになったように見えるんだけど mvn はなんもエラーはいてないぞ、と。

**実行例）**

```text:コマンド実行例
$ sh -x install.sh
(省略)
+ cd /home/ssm-user/aws-saas-boost/installer
+ echo 'Build installer jar with maven'
Build installer jar with maven
+ mvn
+ echo 'Error with build of Java installer for SaaS Boost'
Error with build of Java installer for SaaS Boost
+ exit 2
```

そこでスクリプトの中身を見てみたんです。すると、mvn の標準出力も標準エラー出力も/dev/null にリダイレクトしてましたよ、と。

**実行例）**

```text:コマンド実行例
$ less install.sh
(/mvn)
(n)
if ! mvn > /dev/null 2>&1 ;
then
  echo "Error with build of Java installer for SaaS Boost"
  exit 2
fi
```

ということで、mvn コマンドが何を出力するのかを見たくて手動で実行しましたよ、と。
以下のようにちゃんとメッセージでてくれました。「javac: invalid target release: 11」か、、、Java のターゲットリリースが 11 じゃないと怒られてた^^;

**実行例）**

```text:コマンド実行例
$ cd /home/ssm-user/aws-saas-boost/installer
$ mvn
(省略)
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.6.0:compile (default-compile) on project SaaSBoostInstall: Compilation failure
[ERROR] javac: invalid target release: 11
[ERROR] Usage: javac <options> <source files>
[ERROR] use -help for a list of possible options
[ERROR] -> [Help 1]
[ERROR]
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR]
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
```

確認すると、たしかに OpenJDK のバージョン 8 が設定されておりました。凡ミス。

**実行例）**

```text:コマンド実行例
$ java -version
openjdk version "1.8.0_282"
$ mvn -version
Java version: 1.8.0_282, vendor: Red Hat, Inc.
```

ということで、Corretto のバージョン 11 で動くように設定して、再度 mvn を実行すると成功しましたよ、と。

**実行例）**

```text:コマンド実行例
$ sudo alternatives --config java
$ java -version
openjdk version "11.0.11" 2021-04-20 LTS
$ export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto.x86_64/
$ mvn -version
Java version: 11.0.11, vendor: Amazon.com Inc.
$ mvn
```

ということで install.sh から実行してもちゃんとビルドは通りました。ちゃんちゃん。

### Case2: 「Could not execute 'aws sts get-caller-identity', please check AWS CLI configuration.」というエラーメッセージを出力してスクリプト停止

でも、すぐに次のエラーはやってきんです。「aws sts get-caller-identity」の実行ができなかったぞ、と。

**実行例）**

```text:コマンド実行例
$ sh -x install.sh
：
+ echo 'Launch Java Installer for SaaS Boost'Launch Java Installer for SaaS Boost
+ java -Djava.util.logging.config.file=logging.properties -jar /home/ssm-user/aws-saas-boost/installer/target/SaaSBoostInstall-1.0.0-shaded.jar
WARNING: sun.reflect.Reflection.getCallerClass is not supported. This will impact performance.
Setting version to v0 as it is missing from the git properties file.
===========================================================
Welcome to the AWS SaaS Boost Installer
Setting version to v0 as it is missing from the git properties file.
Installer Version: d64277c-dirty, Commit time: 2021-05-19T21:15:02+0000
Checking maven, yarn and AWS CLI...
Could not execute 'aws sts get-caller-identity', please check AWS CLI configuration.
```

実行できなかったってどういう意味なのかさっぱり分からないので、まずは手動で実行してみるとあっさり原因判明。YAML という知らない出力タイプが設定されているって言われてました、、、なるほど aws configure の設定値が悪いのか。

**実行例）**

```text:コマンド実行例
$ aws sts get-caller-identity
Unknown output type: yaml
```

ということで出力フォーマットを JSON 形式にして実行すると、ちゃんとインストーラのメニュー表示まで進んでくれました。ちゃんちゃん。

**実行例）**

```text:コマンド実行例
$ aws configure set output json
$ sh install.sh
：
===========================================================
Welcome to the AWS SaaS Boost Installer
Setting version to v0 as it is missing from the git properties file.
Installer Version: d64277c-dirty, Commit time: 2021-05-19T21:15:02+0000
Checking maven, yarn and AWS CLI...
Environment Checks for maven, yarn, and AWS CLI PASSED.
===========================================================
1. New AWS SaaS Boost install.
2. Install Metrics and Analytics in to existing AWS SaaS Boost deployment.
3. Update Web Application for existing AWS SaaS Boost deployment.
4. Update existing AWS SaaS Boost deployment.
5. Delete existing AWS SaaS Boost deployment.
6. Exit installer.
Please select an option to continue (1-6):
```

ということでエラー集のご紹介でした。

## では改造して使ってみよう

なんですが、もうオリジナルを導入しただけおなかいっぱいだな、、、ってことで、またどこか時間が取れるときにでも更新しようと思います^^;

(まぁ後回しにした場合は大抵やらないんだけどね、、、)

## 終わりに

ということで、今回は「AWS SaaS Boost がオープンソースとして公開されたのでさっそくいじってみよう（と思っただけ）」でした。

、、、えっ！？タイトルが違う？まぁいいじゃないかｗ
