---
title: "AWS Controllers for Kubernetesを使って各種AWSサービスをマニフェストファイルで管理しよう"
date: 2022-02-16
tags: ["AWS", "Amazon EKS", "Kubernetes", "Amazon S3", "Amazon ECR"]
draft: false
---

みなさん、こんにちは。今回はさまざまなAWSサービスをKubernetesから管理できるようにするAWS Controllers for Kubernetes(ACK)のお話です。

みなさんはAmazon EKSを活用してKubernetesクラスタをAWS上で動かすとなった際に、他のマネージドサービスの利用はどうされていますか。もちろんすべてKubernetes上で動かしてシステムを完結させるという選択肢もあるかと思いますが、やはり多くの方が他のAWSのマネージドサービスの併用も検討されるのではないでしょうか。その一方で、これら併用環境のコード化 (IaC、Infrastructure as Code) を実現しようとすると、Kubernetesアプリケーションの管理はHelm、AWSリソースの管理はTerraform、などという別々のツールでの管理になってしまいがちです。

そんな悩みを解決する1つの手段がAWS Load Balancer ControllerやAWS Controllers for KubernetesといったKubernetesクラスタ機能を拡張する各種コントローラの活用です。これらのコントローラを利用することで、AWSリソースについてもKubernetesマニフェストファイルで定義できるようになり、Kubernetes側に運用管理を寄せてシンプル化することが可能です。

今回はそのうちの1つ、さまざまなAWSサービスを管理できるようにするAWS Controllers for Kubernetesについて、簡単なサンプルを交えて紹介していきたいと思います。これからAmazon EKS上にアプリケーションを展開しようと考えている方は参考にしてみてはいかがでしょうか。

## AWS Controllers for Kubernetes(ACK)とは

AWS Controllers for Kubernetes(ACK)は、さまざまなAWSサービスをKubernetesクラスタから管理するためのKubernetes API拡張コントローラ群の総称です。このコントローラ群を活用することで、Kubernetesクラスタから直接AWSサービスの定義、作成を行うことが可能になり、アプリケーションとその依存関係にあるデータベース、メッセージキュー、オブジェクトストレージなどのマネージドサービスを含むすべてをKubernetesにて一元管理することが可能となります。なお、現時点のACKでは、次のAWSサービス向けコントローラがディベロッパープレビュー機能として利用可能となっています。

- Amazon API Gateway V2
- Amazon Application Auto Scaling
- Amazon DynamoDB
- Amazon ECR
- Amazon EKS
- Amazon ElastiCache
- Amazon EC2
- Amazon MQ
- Amazon OpenSearch Service
- Amazon RDS
- Amazon SageMaker
- Amazon SNS
- AWS Step Functions
- Amazon S3

https://github.com/aws-controllers-k8s/community

## ACKを導入してみよう

### Step1. 作業環境の設定

今回はAmazon EKSやACKの管理を行う環境としてAWS CloudShellを利用していきたいと思います。まずは操作に必要な各種ツールの設定をAWS CloudShellにしてきましょう。

#### #1. AWS CLIの設定

AWSリソースの操作を行えるように次のコマンドを実行し、AWS CLIの設定を行いましょう。

**実行例）AWS CLIの設定**

```bash
aws configure
```

#### #2. kubectlコマンドのインストール

次にKubernetes管理ツールの`kubectl`コマンドをインストールしましょう。

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

```txt
$ kubectl version --short --client
Client Version: v1.21.2-13+d2965f0db10712
```

#### #3. eksctlコマンドのインストール

続いてAmazon EKS管理ツールの`eksctl`コマンドをインストールしましょう。

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
```txt
$ eksctl version
0.83.0
```

#### #4. helmコマンドのインストール

最後にKubernetes上で稼働するアプリケーションを管理するためのツールである`helm`コマンドをインストールしましょう。

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

```txt
$ helm version --short
v3.8.0+gd141386
```

以上で作業環境(AWS CloudShell)の設定は完了です。

### Step2. EKSクラスタの作成

続いてACKを導入する対象のAmazon EKSクラスタを作成していきましょう。今回はあくまで検証なので`eksctl`コマンドでサクッと作成していきましょう。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/getting-started-eksctl.html

**実行例）EKSクラスタの作成**

```bash
# 環境変数の設定
export CLUSTER="matt-tokyo-cluster"

# EKS クラスタの作成
eksctl create cluster --name ${CLUSTER} --version 1.21

# サービスアカウントでの IAM ロール使用を許可
eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER} --approve
```

ここまで終わりましたらACKを利用するための事前準備は完了です。

### Step3. コントローラのデプロイ

それではACKをAmazon EKSクラスタにデプロイしていきましょう。Amazon ECRパブリックギャラリーにて各AWSサービスに対応するコントローラ用Helmチャートが公開されておりますのでこちらを利用してデプロイしていきたいと思います。

https://gallery.ecr.aws/aws-controllers-k8s

#### #1. インストールスクリプトの作成

ACKは各AWSサービスに対応するコントローラをそれぞれ導入し、サービスアカウントの設定をしていく必要があるのですが、公式ドキュメントを見るとかなり煩雑な手順になっていることがわかります。そこでコマンド1つでインストールできるようにスクリプト化してみました。

https://aws-controllers-k8s.github.io/community/docs/user-docs/install

{{< alert >}}
再実行を想定してない作りになっているのでご注意ください。
{{< /alert >}}


**作成例）install.sh**

```bash
#!/bin/bash
set -eux

# 引数の確認
if [ $# -eq 0  ] && [ -z "${SERVICE}" ]; then
    echo "Error: usage: ./install.sh <SERVICE_NAME>"
    exit 1
elif [ $# -eq 1  ] && [ -n "$1" ]; then
    SERVICE=$1
fi

# 環境変数の設定
export HELM_EXPERIMENTAL_OCI=1
CHART_EXPORT_PATH="/tmp/chart"
ACK_K8S_NAMESPACE=${ACK_K8S_NAMESPACE:-"ack-system"}
ACK_SERVICE_CONTROLLER="ack-${SERVICE}-controller"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
CLUSTER=${CLUSTER:-"my-tokyo-cluster"}
OIDC_PROVIDER=$(aws eks describe-cluster --name ${CLUSTER} \
    --query "cluster.identity.oidc.issuer" --output text \
    | sed -e "s/^https:\/\///")

# 最新リリースバージョン名の取得
RELEASE_VERSION=$(curl -sL \
    https://api.github.com/repos/aws-controllers-k8s/${SERVICE}-controller/releases/latest \
    | grep '"tag_name":' | cut -d'"' -f4)

if [ -z "${RELEASE_VERSION}" ]; then
    # latestリリースが作成されていない場合は一覧から最新バージョンを取得
    RELEASE_VERSION=$(curl -sL \
        https://api.github.com/repos/aws-controllers-k8s/${SERVICE}-controller/releases \
        | grep '"tag_name":' | head -n 1 | cut -d'"' -f4)

    # リリースを何も作成されていない場合はv0.0.1を設定(SNSコントローラ向け)
    RELEASE_VERSION=${RELEASE_VERSION:-v0.0.1}
fi

# Helmチャートのダウンロードディレクトリの作成
mkdir -p ${CHART_EXPORT_PATH}

# Helmチャートのダウンロード
helm pull oci://public.ecr.aws/aws-controllers-k8s/${SERVICE}-chart \
    --version $RELEASE_VERSION -d ${CHART_EXPORT_PATH}

# アーカイブの解凍
tar xvf ${CHART_EXPORT_PATH}/${SERVICE}-chart-${RELEASE_VERSION}.tgz \
    -C ${CHART_EXPORT_PATH}

# 解凍後のパスチェック
if [ -d ${CHART_EXPORT_PATH}/${SERVICE}-chart ]; then
    CHART_PATH=${CHART_EXPORT_PATH}/${SERVICE}-chart
else
    # SNSコントローラ向け
    CHART_PATH=${CHART_EXPORT_PATH}/${ACK_SERVICE_CONTROLLER}
fi

# コントローラのインストール
helm install --create-namespace ${ACK_SERVICE_CONTROLLER} \
    --namespace ${ACK_K8S_NAMESPACE} \
    --set aws.region="${AWS_REGION}" ${CHART_PATH}

# IAMロール定義ファイルの作成
cat <<EOF > /tmp/${ACK_SERVICE_CONTROLLER}.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:${ACK_K8S_NAMESPACE}:${ACK_SERVICE_CONTROLLER}"
        }
      }
    }
  ]
}
EOF

# IAMロールの作成
aws iam create-role \
    --role-name ${ACK_SERVICE_CONTROLLER} \
    --assume-role-policy-document file:///tmp/${ACK_SERVICE_CONTROLLER}.json

# 推奨IAMポリシーの設定
POLICY_ARN_STRINGS=$(curl -sL \
    https://raw.githubusercontent.com/aws-controllers-k8s/${SERVICE}-controller/main/config/iam/recommended-policy-arn)

if [ "404: Not Found" != "${POLICY_ARN_STRINGS}" ]; then
    while IFS= read -r POLICY_ARN; do
        aws iam attach-role-policy \
            --role-name "${ACK_SERVICE_CONTROLLER}" \
            --policy-arn "${POLICY_ARN}"
    done <<< "${POLICY_ARN_STRINGS}"
fi

# 推奨IAMポリシーの設定(EKSコントローラ向け)
INLINE_POLICY=$(curl -sL \
    https://raw.githubusercontent.com/aws-controllers-k8s/${SERVICE}-controller/main/config/iam/recommended-inline-policy)

if [ "404: Not Found" != "${INLINE_POLICY}" ]; then
    aws iam put-role-policy \
        --role-name "${ACK_SERVICE_CONTROLLER}" \
        --policy-name "ack-recommended-policy" \
        --policy-document "${INLINE_POLICY}"
fi

# サービスアカウントにIAMロールを関連付け
kubectl -n ${ACK_K8S_NAMESPACE} annotate serviceaccount \
    ${ACK_SERVICE_CONTROLLER} \
    eks.amazonaws.com/role-arn=$(aws iam get-role \
        --role-name=${ACK_SERVICE_CONTROLLER} --query Role.Arn --output text)

# Deploymentリソースを再起動
kubectl -n ${ACK_K8S_NAMESPACE} rollout restart deployment \
    $(kubectl -n ${ACK_K8S_NAMESPACE} get deployment \
        -o custom-columns=Name:metadata.name --no-headers \
        | grep ${ACK_SERVICE_CONTROLLER})
```

#### #2. コントローラのインストール

それではコントローラを導入していきたいと思います。本記事では例としてAmazon Simple Storage Service(S3)サービスに対応したコントローラをインストールしていきたいと思います。

**実行例）コントローラのインストール(S3の場合)**

```bash
# 環境変数の設定
export SERVICE="s3"
export ACK_K8S_NAMESPACE="ack-system"
export ACK_SERVICE_CONTROLLER="ack-${SERVICE}-controller"

# コントローラのインストール
bash install.sh

# 確認
helm list -n ${ACK_K8S_NAMESPACE} -o yaml -f ${ACK_SERVICE_CONTROLLER}
```

インストールに成功していれば出力例のようにコントローラが表示されるようになります。

**出力例）**

```txt
$ helm list -n ${ACK_K8S_NAMESPACE} -o yaml -f ${ACK_SERVICE_CONTROLLER}
- app_version: v0.0.13
  chart: s3-chart-v0.0.13
  name: ack-s3-controller
  namespace: ack-system
  revision: "1"
  status: deployed
  updated: 2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC
```

なお、今回紹介したS3以外のコントローラを導入する際は、環境変数`SERVICE`の値を各 AWS サービスに対応する文字列へ置換することでインストールすることが可能です。各サービスに対応する文字列は次の通りです。

|AWS サービス名|SERVICE 値|
|---|---|
| Amazon API Gateway V2           |apigatewayv2|
| Amazon Application Auto Scaling |applicationautoscaling|
| Amazon DynamoDB                 |dynamodb|
| Amazon ECR                      |ecr|
| Amazon EKS                      |eks|
| Amazon ElastiCache              |elasticache|
| Amazon EC2                      |ec2|
| Amazon MQ                       |mq|
| Amazon OpenSearch Service       |opensearchservice|
| Amazon RDS                      |rds|
| Amazon SageMaker                |sagemaker|
| Amazon SNS                      |sns|
| AWS Step Functions              |sfn|
| Amazon S3                       |s3|

ちなみに、全部入れるとこのような感じでコントローラだらけになってしまいます^^;

**出力例）**

```txt
$ helm list -n ${ACK_K8S_NAMESPACE}
NAME                                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                                   APP VERSION
ack-apigatewayv2-controller             ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        apigatewayv2-chart-v0.0.15              v0.0.15    
ack-applicationautoscaling-controller   ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        applicationautoscaling-chart-v0.2.4     v0.2.4     
ack-dynamodb-controller                 ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        dynamodb-chart-v0.0.14                  v0.0.14    
ack-ec2-controller                      ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        ec2-chart-v0.0.7                        v0.0.7     
ack-ecr-controller                      ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        ecr-chart-v0.0.19                       v0.0.19    
ack-eks-controller                      ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        eks-chart-v0.0.8                        v0.0.8     
ack-elasticache-controller              ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        elasticache-chart-v0.0.14               v0.0.14    
ack-mq-controller                       ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        mq-chart-v0.0.12                        v0.0.12    
ack-opensearchservice-controller        ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        opensearchservice-chart-v0.0.9          v0.0.9     
ack-rds-controller                      ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        rds-chart-v0.0.17                       v0.0.17    
ack-s3-controller                       ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        s3-chart-v0.0.13                        v0.0.13    
ack-sagemaker-controller                ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        sagemaker-chart-v0.3.0                  v0.3.0     
ack-sfn-controller                      ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        sfn-chart-v0.0.11                       v0.0.11    
ack-sns-controller                      ack-system      1               2022-02-xx xx:xx:xx.xxxxxxxxx +0000 UTC deployed        ack-sns-controller-v0.0.1               v0.0.1
```

## ACKを使ってみよう

今回は一部のAWSサービス向けコントローラを簡単なサンプルを用いて使い方を紹介していきたいと思います。すべてのコントローラの使い方については公式ドキュメントを参照ください。

https://aws-controllers-k8s.github.io/community/reference/

### Amazon ECRコントローラの使い方

ACK環境では、次のサンプルのようにKubernetes Repositoryリソースを定義をすることによってECRリポジトリをプロビジョニングできます。定義可能なスペックの詳細については次の公式リファレンスドキュメントを参照ください。

https://aws-controllers-k8s.github.io/community/reference/ecr/v1alpha1/repository/

**作成例）repository.yaml (ECRリポジトリリソース定義サンプル)**

```yaml
apiVersion: ecr.services.k8s.aws/v1alpha1
kind: Repository
metadata:
  name: "matt-ecr-repository"
spec:
  name: "matt-ecr-repository"
  imageScanningConfiguration:
    scanOnPush: true
```

それでは次のコマンドを実行してサンプルECRリポジトリリソースをデプロイしていきましょう。

**実行例）サンプルECRリポジトリのデプロイ**

```bash
export NAMESPACE="sample"

# サンプルECRリポジトリ用Namespaceの作成
kubectl create namespace ${NAMESPACE}

# サンプルECRリポジトリのデプロイ
kubectl apply -n ${NAMESPACE} -f repository.yaml

# サンプルECRリポジトリがデプロイされたことを確認
kubectl get -n ${NAMESPACE} repository
```

ECRリポジトリリソースのデプロイに成功した場合は次の出力例のようにRepositoryリソース一覧に表示されます。

**出力例）**

```txt
$ kubectl get -n ${NAMESPACE} repository
NAME                  AGE
matt-ecr-repository   10s
```

念のため、AWS CLIを実行してECRリポジトリが作成されているかを確認してみましょう。

**実行例）ECRリポジトリの確認**

```bash
aws ecr describe-repositories --repository-name matt-ecr-repository
```

AWS CLIでもECRリポジトリが確認できたら成功です。

**出力例）**

```txt
$ aws ecr describe-repositories --repository-name matt-ecr-repository
repositories:
- createdAt: '2022-02-xxTxx:xx:xx+00:00'
  encryptionConfiguration:
    encryptionType: AES256
  imageScanningConfiguration:
    scanOnPush: true
  imageTagMutability: MUTABLE
  registryId: 'xxxxxxxxxxxx'
  repositoryArn: arn:aws:ecr:ap-northeast-1:xxxxxxxxxxxx:repository/matt-ecr-repository
  repositoryName: matt-ecr-repository
  repositoryUri: xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/matt-ecr-repository
```

最後にサンプルを削除して動作確認は終了です。お疲れ様でした。

**実行例）サンプルの削除**

```bash
kubectl delete namespace ${NAMESPACE}
```

### Amazon S3コントローラの使い方

ACK環境では、次のサンプルのようにKubernetes Bucketリソースを定義をすることによってS3バケットをプロビジョニングできます。定義可能なスペックの詳細については次の公式リファレンスドキュメントを参照ください。

https://aws-controllers-k8s.github.io/community/reference/s3/v1alpha1/bucket/

**作成例）bucket.yaml (S3バケットリソース定義サンプル)**

```yaml
apiVersion: s3.services.k8s.aws/v1alpha1
kind: Bucket
metadata:
  name: "matt-s3-bucket"
spec:
  name: "matt-s3-bucket"
  publicAccessBlock:
    blockPublicACLs: true
    blockPublicPolicy: true
  versioning:
    status: Enabled
  encryption:
    rules:
    - bucketKeyEnabled: false
      applyServerSideEncryptionByDefault:
        sseAlgorithm: AES256
```

それでは次のコマンドを実行してサンプルS3バケットリソースをデプロイしていきましょう。

**実行例）サンプルS3バケットのデプロイ**

```bash
export NAMESPACE="sample"

# サンプルS3バケット用Namespaceの作成
kubectl create namespace ${NAMESPACE}

# サンプルS3バケットのデプロイ
kubectl apply -n ${NAMESPACE} -f bucket.yaml

# サンプルS3バケットがデプロイされたことを確認
kubectl get -n ${NAMESPACE} bucket
```

S3バケットリソースのデプロイに成功した場合は次の出力例のようにBucketリソース一覧に表示されます。

**出力例）**

```txt
$ kubectl get -n ${NAMESPACE} bucket
NAME             AGE
matt-s3-bucket   1m22s
```

念のため、AWS CLIを実行してS3バケットが作成されているかを確認してみましょう。

**実行例）S3バケットが作成されているかを確認**

```bash
aws s3 ls | grep "matt-s3-bucket"
```

AWS CLIでもS3バケットが確認できたら成功です。

**出力例）**

```txt
$ aws s3 ls | grep "matt-s3-bucket"
2022-02-xx xx:xx:xx matt-s3-bucket
```

最後にサンプルを削除して動作確認は終了です。お疲れ様でした。

**実行例）サンプルの削除**

```bash
kubectl delete namespace ${NAMESPACE}
```

## 終わりに

今回はさまざまなAWSサービスをAmazon Elastic Kubernetes Service(EKS)上で管理できるようにするAWS Controllers for Kubernetesのご紹介でしたがいかがだったでしょうか。

現時点ではディベロッパープレビュー機能のためプロダクション用途での活用はまだまだオススメできませんが、もしAWS Controllers for Kubernetesの活用を検討されている方は参考にしていただければ幸いです。

以上、さまざまなAWSサービスをKubernetesから管理できるようにする「AWS Controllers for Kubernetes」のご紹介でした。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Kubernetes は、The Linux Foundation の米国およびその他の国における登録商標または商標です。
- その他、本資料に記述してある会社名、製品名は、各社の登録商品または商標です。
