---
title: "AWS Load Balancer Controller を使って ELB を Kubernetes のマニフェストファイルで管理しよう"
date: 2021-12-14
tags: ["AWS", "Kubernetes", "Amazon EKS", "Elastic Load Balancing(ELB)"]
draft: false
---

みなさん、こんにちは。今回は Amazon Elastic Kubernetes Service(EKS) を利用する際に併せて利用したい AWS Load Balancer Controller のお話です。

みなさんは Amazon EKS を活用して Kubernetes クラスタを AWS 上で動かすとなった際に、他のマネージドサービスの利用はどうされていますか。もちろんすべて Kubernetes 上で動かしてシステムを完結させるという選択肢もあるかと思いますが、やはり多くの方が他の AWS のマネージドサービスの併用も検討されるのではないでしょうか。その一方で、これら併用環境のコード化 (IaC、Infrastructure as Code) を実現しようとすると、Kubernetes アプリケーションの管理は Helm で、AWS リソースの管理は Terraform で、などという別々のツールでの管理になってしまいがちです。

そんな悩みを解決する一つの手段が AWS Load Balancer Controller や AWS Controllers for Kubernetes といった Kubernetes クラスタ機能を拡張する各種コントローラの活用です。これらのコントローラを利用することで、AWS リソースについても Kubernetes マニフェストファイルで定義できるようになり、Kubernetes 側に運用管理を寄せてシンプル化することができます。

今回はそのうちの一つ、Elastic Load Balancing(ELB) を Kubernetes クラスタで管理できるようにする AWS Load Balancer Controller について、簡単なサンプルアプリケーションを交えて紹介していきたいと思います。これから Amazon EKS 上にアプリケーションを展開しようと考えている方は参考にしてみてはいかがでしょうか。

## AWS Load Balancer Controller とは

AWS Load Balancer Controller (旧AWS ALB Ingress Controller) は、ELB を Kubernetes クラスタから管理するためのコントローラです。このコントローラを活用することで、Kubernetes Ingress リソースとして L7 ロードバランサの Application Load Balancer(ALB) を、Kuberntes Service リソースとして L4 ロードバランサの Network Load Balancer(NLB) を利用することができるようになります。

Kubernetes Service/Ingress リソースでの処理を外部ロードバランサである ELB へ切り出すことによって、ワークロードへの性能影響の低減、ノードリソースの利用効率の向上、Service/Ingress リソースのスケーリングなどを AWS 側へ任せることで運用負荷の低減といったことができる見込みです。

https://github.com/kubernetes-sigs/aws-load-balancer-controller

## AWS Load Balancer Controller を導入してみよう

### Step1. 作業環境の設定

今回は Amazon EKS や AWS Load Balancer Controller の管理を行う環境として AWS CloudShell を利用していきたいと思います。まずは操作に必要な各種ツールの設定を AWS CloudShell にしてきましょう。

#### AWS CLI の設定

AWS リソースの操作を行えるように次のコマンドを実行し、AWS CLI の設定を行いましょう。

**実行例）AWS CLIの設定**

```bash
aws configure
```

#### kubectl コマンドのインストール

次に Kubernetes 管理ツールの `kubectl` コマンドをインストールしましょう。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/install-kubectl.html#linux

**実行例）kubectlコマンドのインストール**

```bash
# kubectl コマンドのダウンロード
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl

# 実行権限の付与
chmod +x kubectl

# 実行ファイルのパスを設定
mkdir -p ${HOME}/bin && mv kubectl ${HOME}/bin && export PATH=${PATH}:${HOME}/bin

# シェルの起動時に $HOME/bin をパスへ追加
echo 'export PATH=${PATH}:${HOME}/bin' >> ~/.bashrc

# インストールが成功していることを確認
kubectl version --short --client
```

インストールに成功していれば出力例のようにバージョン情報の出力を確認できます。

**出力例）**

```
$ kubectl version --short --client
Client Version: v1.21.2-13+d2965f0db10712
```

#### eksctl コマンドのインストール

続いて Amazon EKS 管理ツールの `eksctl` コマンドをインストールしましょう。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/eksctl.html#linux

**実行例）eksctlコマンドのインストール**

```bash
# eksctl の最新バージョンをダウンロード
curl -L "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# 実行ファイルをパスの通ったディレクトリへ移動
mv /tmp/eksctl ${HOME}/bin

# インストールが成功していることを確認
eksctl version
```

インストールに成功していれば出力例のようにバージョン情報の出力を確認できます。

**出力例）**

```
$ eksctl version
0.76.0
```

#### helm コマンドのインストール

最後に Kubernetes 上で稼働するアプリケーションを管理するためのツールである `helm` コマンドをインストールしましょう。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/helm.html

**実行例）helmコマンドのインストール**

```bash
# 前提パッケージのインストール
sudo yum install -y openssl

# インストールスクリプトのダウンロード
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh

# 実行権限の付与
chmod 700 get_helm.sh

# インストールスクリプトの実行
./get_helm.sh

# インストールが成功していることを確認
helm version --short
```

インストールに成功していれば出力例のようにバージョン情報の出力を確認できます。

**出力例）**

```
$ helm version --short
v3.7.2+g663a896
```

以上で作業環境(AWS CloudShell)の設定は完了です。

### Step2. EKS クラスタの作成

続いて AWS Load Balancer Controller を導入する対象の Amazon EKS クラスタを作成していきましょう。今回は Kubernetes ノードには AWS Fargate を使用していきたいと思います。それでは `eksctl` コマンドを実行してクラスタを作成しましょう。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/getting-started-eksctl.html

**実行例）EKSクラスタの作成**

```bash
# 環境変数の設定
export CLUSTER="my-tokyo-cluster"

# EKS クラスタの作成
eksctl create cluster --name ${CLUSTER} --version 1.21 --fargate

# サービスアカウントでの IAM ロール使用を許可
eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER} --approve
```

ここまで終わりましたら AWS Load Balancer Controller を利用するための事前準備は完了です。

### Step3. コントローラのデプロイ

それでは AWS Load Balancer Controller を Amazon EKS クラスタにデプロイしていきましょう。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/aws-load-balancer-controller.html

#### サービスアカウントの作成

まずは AWS Load Balancer Controller 用のサービスアカウントの作成を行っていきます。今回は `kube-system` Namespace に `aws-load-balancer-controller` という名前でサービスアカウントを作成して行きたいと思います。それでは次のコマンドを実行してサービスアカウントを作成しましょう。

**実行例）サービスアカウントの作成**

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
POLICY_NAME="myAWSLoadBalancerControllerIAMPolicy"
SERVICE_ACCOUNT="aws-load-balancer-controller"

# IAM ポリシー定義ファイルのダウンロード
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# IAM ポリシーの作成
aws iam create-policy \
    --policy-name ${POLICY_NAME} \
    --policy-document file://iam_policy.json

# サービスアカウントの作成
eksctl create iamserviceaccount \
  --name=${SERVICE_ACCOUNT} \
  --cluster=${CLUSTER} \
  --namespace="kube-system" \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${POLICY_NAME} \
  --override-existing-serviceaccounts \
  --approve
```

#### コントローラのインストール

次に AWS Load Balancer Controller をインストールしていきましょう。なお、AWS Load Balancer Controller のインストール方法はいくつか用意されていますが、AWS Fargate 環境の場合は Helm を利用したインストール方法を選択する必要があるためご注意ください。

**実行例）コントローラのインストール**

```bash
# EKS 用の Helm レポジトリを追加
helm repo add eks https://aws.github.io/eks-charts

# TargetGroupBinding カスタムリソースをインストール
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

# Helm チャートからインストール
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --set clusterName=${CLUSTER} \
    --set serviceAccount.create=false \
    --set vpcId=$(aws cloudformation describe-stacks \
        --stack-name "eksctl-${CLUSTER}-cluster" \
        --query 'Stacks[0].Outputs[?OutputKey==`VPC`].OutputValue' \
        --output text) \
    --set serviceAccount.name=${SERVICE_ACCOUNT} \
    -n kube-system

# 確認
kubectl get deployment -n kube-system aws-load-balancer-controller
```

インストールに成功していれば出力例のようにコントローラが一覧に表示されるようになります。

**出力例）**

```
$ kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           42s
```

以上で AWS Load Balancer Controller の導入は完了です。

## AWS Load Balancer Controller を使ってみよう

### Network Load Balancer (Serviceリソース) の使い方

AWS Load Balancer Controller 環境では、次のサンプルのように Kubernetes Service リソース定義にて「**LoadBalancer タイプの指定**」と「**アノテーションとして各種パラメータを指定**」をすることによって Network Load Balancer(NLB) をプロビジョニングすることができます。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/network-load-balancing.html

**作成例）Serviceリソース定義サンプル**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
spec:
  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
  selector:
    app: sample-app
```

アノテーションに指定する主要なパラメータとしては次の通りです。アノテーションに指定可能なパラメータの一覧につきましては次の URL を参照してください。

https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/service/annotations/

|アノテーション名|説明|
|---|---|
|service.beta.kubernetes.io/aws-load-balancer-type|ロードバランサのプロビジョニングに使用するコントローラを指定します。AWS Load Balancer Controller の場合は "**external**" を指定します。|
|service.beta.kubernetes.io/aws-load-balancer-scheme|ロードバランサのスキームを指定します。内部向けの場合は "**internal**"、インターネット向けの場合は "**internet-facing**" を指定します。|
|service.beta.kubernetes.io/aws-load-balancer-nlb-target-type|バックエンドに指定するターゲットタイプを指定します。トラフィックを直接 Pod にルーティングするには "**ip**" を指定します。|
|service.beta.kubernetes.io/aws-load-balancer-healthcheck-port|バックエンドのヘルスチェック用ポート("**traffic-port**"/"**ポート番号**")を指定します。デフォルトは "**traffic-port**" です。|
|service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol|バックエンドのヘルスチェック用プロトコル("**tcp**"/"**http**"/"**https**")を指定します。デフォルトは "**tcp**" です。|
|service.beta.kubernetes.io/aws-load-balancer-healthcheck-path|バックエンドの HTTP/HTTPS ヘルスチェック用パスを指定します。デフォルトは "**/**" です。|

#### NLB Service のサンプル

それではサンプルアプリケーションをデプロイして NLB Service リソースを実際に動かしてみましょう。今回は次のサンプルアプリケーションをベースに少しだけ手を加えたものを用いて動作確認をしていきたいと思います。

https://github.com/istio/istio/tree/master/samples/helloworld

**作成例）helloworld-nlb-sample.yaml (サンプルアプリケーション定義ファイル)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: /health
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
    name: http
  selector:
    app: helloworld
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v1
  labels:
    app: helloworld
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v1
  template:
    metadata:
      labels:
        app: helloworld
        version: v1
    spec:
      containers:
      - name: helloworld
        image: docker.io/istio/examples-helloworld-v1
        ports:
        - containerPort: 5000
```

それでは次のコマンドを実行してサンプルアプリケーションをデプロイしていきましょう。

**実行例）サンプルアプリケーションのデプロイ**

```bash
export FARGATEPROFILE="sample-app"
export NAMESPACE="sample"

# Fargate プロファイルの作成
eksctl create fargateprofile --cluster ${CLUSTER} --name ${FARGATEPROFILE} --namespace ${NAMESPACE}

# サンプルアプリケーション用 Namespace の作成
kubectl create namespace ${NAMESPACE}

# サンプルアプリケーションのデプロイ
kubectl apply -n ${NAMESPACE} -f helloworld-nlb-sample.yaml

# サービスがデプロイされたことを確認
kubectl get -n ${NAMESPACE} service helloworld
```

NLB Service のデプロイに成功した場合は次の出力例のように Service リソース一覧に表示されます。

**出力例）**

```
$ kubectl get -n ${NAMESPACE} service helloworld
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP                                                                        PORT(S)        AGE
helloworld   LoadBalancer   10.100.114.79   k8s-sample-hellowor-xxxxxxxxxx-xxxxxxxxxxxxxxxx.elb.ap-northeast-1.amazonaws.com   80:32642/TCP   32h
```

NLB のヘルスチェックが正常になるまで数分待った後、NLB のパブリックエンドポイントにアクセスしてみましょう。

**実行例）サンプルアプリケーションへアクセス**

```bash
# NLB の DNS 名を取得
ENDPOINT=$(kubectl get -n ${NAMESPACE} service helloworld \
    -o custom-columns=HOSTNAME:status.loadBalancer.ingress[0].hostname \
    --no-headers)

# テストの実行
curl http://${ENDPOINT}/hello
```

期待通りのレスポンスが返ってくることが確認できたら成功です。

**出力例）**

```txt
$ curl http://${ENDPOINT}/hello
Hello version: v1, instance: helloworld-v1-b9d9d6679-hdqsq
```

最後にサンプルアプリケーションを削除して動作確認は終了です。お疲れ様でした。

**実行例）サンプルアプリケーションの削除**

```bash
kubectl delete namespace ${NAMESPACE}
```

### Application Load Balancer (Ingressリソース) の使い方

AWS Load Balancer Controller 環境では、次のサンプルのように「**Kubernetes IngressClass リソースにてコントローラの指定**」を、「**Kubernetes Ingress リソース定義のアノテーションとして各種パラメータを指定**」をすることによって Application Load Balancer(ALB) をプロビジョニングすることができます。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/alb-ingress.html

{{< alert >}}
Kubernetes Ingress リソースのアノテーションに `kubernetes.io/ingress.class: alb` を指定することでも作成できますが、Kubernetes 1.18 以降は  `kubernetes.io/ingress.class` の利用は非推奨となっておりますのでご注意ください。
{{< /alert >}}

**作成例）Ingressリソース定義サンプル**

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: sample-ingress-class
spec:
  controller: ingress.k8s.aws/alb
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: sample-ingress-class
  rules:
  - http:
      paths:
      - path: /hello
        pathType: Exact
        backend:
          service:
            name: sample-service
            port:
              number: 80
```

アノテーションに指定する主要なパラメータとしては次の通りです。なお、ALB Ingress は AWS WAF や AWS Shield による脅威からの保護、Amazon Cognito 認証などさまざまなマネージドサービスとの連携が可能です。アノテーションに指定可能なパラメータの一覧につきましては次の URL を参照してください。

https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/

|アノテーション名|説明|
|---|---|
|alb.ingress.kubernetes.io/group.name|複数の Ingress リソースを 1 つの ALB に統合する際にグループ名を指定します。|
|alb.ingress.kubernetes.io/scheme|ロードバランサのスキームを指定します。内部向けの場合は "**internal**"、インターネット向けの場合は "**internet-facing**" を指定します。|
|alb.ingress.kubernetes.io/listen-ports|ロードバランサの受付ポートを指定します。デフォルトは **'[{"HTTP": 80}]' \| '[{"HTTPS": 443}]'** です。|
|alb.ingress.kubernetes.io/target-type|バックエンドに指定するターゲットタイプを指定します。トラフィックを直接 Pod にルーティングするには "**ip**" を指定します。|
|alb.ingress.kubernetes.io/healthcheck-port|バックエンドのヘルスチェック用ポート("**traffic-port**"/"**ポート番号**")を指定します。デフォルトは "**traffic-port**" です。|
|alb.ingress.kubernetes.io/healthcheck-protocol|バックエンドのヘルスチェック用プロトコル("**http**"/"**https**")を指定します。デフォルトは "**http**" です。|
|alb.ingress.kubernetes.io/healthcheck-path|バックエンドの HTTP/HTTPS ヘルスチェック用パスを指定します。デフォルトは "**/**" です。|

#### ALB Ingress のサンプル

それではサンプルアプリケーションをデプロイして ALB Ingress リソースを実際に動かしてみましょう。今回は次のサンプルアプリケーションをベースに少しだけ手を加えたものを用いて動作確認をしていきたいと思います。

https://github.com/istio/istio/tree/master/samples/helloworld

**作成例）helloworld-alb-sample.yaml (サンプルアプリケーション定義ファイル)**

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb-ingress
spec:
  controller: ingress.k8s.aws/alb
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: helloworld
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  ingressClassName: alb-ingress
  rules:
  - http:
      paths:
      - path: /hello
        pathType: Exact
        backend:
          service:
            name: helloworld
            port:
              number: 80
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
    service: helloworld
spec:
  ports:
  - port: 80
    targetPort: 5000
    name: http
  selector:
    app: helloworld
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v1
  labels:
    app: helloworld
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v1
  template:
    metadata:
      labels:
        app: helloworld
        version: v1
    spec:
      containers:
      - name: helloworld
        image: docker.io/istio/examples-helloworld-v1
        ports:
        - containerPort: 5000
```

それでは次のコマンドを実行してサンプルアプリケーションをデプロイしていきましょう。

**実行例）サンプルアプリケーションのデプロイ**

```bash
export FARGATEPROFILE="sample-app"
export NAMESPACE="sample"

# Fargate プロファイルの作成
eksctl create fargateprofile --cluster ${CLUSTER} --name ${FARGATEPROFILE} --namespace ${NAMESPACE}

# サンプルアプリケーション用 Namespace の作成
kubectl create namespace ${NAMESPACE}

# サンプルアプリケーションのデプロイ
kubectl apply -n ${NAMESPACE} -f helloworld-alb-sample.yaml

# Ingress がデプロイされたことを確認
kubectl get -n ${NAMESPACE} ingress helloworld
```

ALB Ingress のデプロイに成功した場合は次の出力例のように Ingress リソース一覧に表示されます。

**出力例）**

```
$ kubectl get -n ${NAMESPACE} ingress helloworld
NAME         CLASS    HOSTS   ADDRESS                                                                    PORTS   AGE
helloworld   <none>   *       k8s-sample-hellowor-xxxxxxxxxx-xxxxxxxx.ap-northeast-1.elb.amazonaws.com   80      70s
```

DNS への登録などが反映されるまで数分待った後、ALB のパブリックエンドポイントにアクセスしてみましょう。

**実行例）サンプルアプリケーションへアクセス**

```bash
# NLB の DNS 名を取得
ENDPOINT=$(kubectl get -n ${NAMESPACE} ingress helloworld \
    -o custom-columns=HOSTNAME:status.loadBalancer.ingress[0].hostname \
    --no-headers)

# テストの実行
curl http://${ENDPOINT}/hello
```

期待通りのレスポンスが返ってくることが確認できたら成功です。

**出力例）**

```txt
$ curl http://${ENDPOINT}/hello
Hello version: v1, instance: helloworld-v1-b9d9d6679-rpfzl
```

最後にサンプルアプリケーションを削除して動作確認は終了です。お疲れ様でした。

**実行例）サンプルアプリケーションの削除**

```bash
kubectl delete namespace ${NAMESPACE}
```

## 終わりに

今回は Amazon Elastic Kubernetes Service(EKS) を利用する際に併せて利用したい AWS Load Balancer Controller のご紹介でしたが、いかがだったでしょうか？

今回は IaC を実現する 1 つの手段として、というカットでの紹介でしたが Kubernetes Ingress/Service リソースを ELB にすることでさまざまなメリットが期待できますので、Amazon EKS をご利用の際は AWS Load Balancer Controller の併用も視野に入れてみてはいかがでしょうか。

なお、今回は紹介しませんでしたが AWS Load Balancer Controller は使いたいけれど、ELB リソースのライフサイクルは Kubernetes からは切り離したいというケースもあるかと思います。そういった場合は AWS Load Balancer Controller の TargetGroupBinding 機能を利用することで実現可能なので参考にしていただければと思います。TargetGroupBinding 機能の詳細については公式ドキュメントを参照してください。

https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/targetgroupbinding/targetgroupbinding/

以上、Kubernetes Service/Ingress リソースと Elastic Load Balancing(ELB) との統合を実現する「AWS Load Balancer Controller」のご紹介でした。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Kubernetes は、The Linux Foundation の米国およびその他の国における登録商標または商標です。
- その他、本資料に記述してある会社名、製品名は、各社の登録商品または商標です。
