---
title: "複数リージョンの GKE クラスタと Anthos Service Mesh でマルチクラスタメッシュ環境を構築してみた"
date: 2021-12-09
tags: ["Google Cloud", "Google Kubernetes Engine(GKE)", "Anthos Service Mesh", "Kubernetes", "Istio"]
draft: false
ShowToc: true
TocOpen: true
---

# はじめに

みなさん、こんにちは。今回は複数のリージョンに展開する各 GKE クラスタを単一の Anthos Service Mesh 環境に追加し、GKE クラスタ間で負荷分散を行う方法についてご紹介していきたいと思います。

複数リージョンの GKE クラスタでマルチクラスタメッシュを構築することにより、予期しない大規模災害の発生にも耐えうる高い可用性と回復力の実現、エンドユーザからより近い位置への振り分けによるレイテンシの改善といったことが期待できるようになる見込みです。

もちろん公式ドキュメントにもマルチクラスタメッシュの構築に関する記載はあるのですが、単にクラスタ間で分散されたことを確認しただけで終わっており、ローカリティを意識したルーティングの設定やメッシュの外からの通信に関する記載はなかったため、今回はここら辺も含めて一気通貫でご紹介したいと思います。もしこれからリージョンをまたがった Anthos Service Mesh 環境の利用を検討している方は参考にしてみてはいかがでしょうか。

# 構築するシステムについて

次の図に示すように限定公開クラスタおよび承認済みネットワーク機能を有効化した複数リージョンの GKE クラスタに対して Anthos Service Mesh (マネージドコントロールプレーン)を導入しています。サービスメッシュ上ではサンプルアプリケーションを動かし、ローカリティを意識した負荷分散についても設定をしていきたいと思います。なお、今回の例では GKE、Anthos Service Mesh のいずれのリリースチャンネルについても安定性重視の Stable チャンネルを採用しています。

![01-architecture.png](images/01-architecture.png)

# それでは構築していきましょう

いつも通り公式ドキュメントを参考にしつつ、公式ドキュメントに書かれていない部分を補足しながら構築をしていきたいと思います。

https://cloud.google.com/service-mesh/docs/unified-install/gke-install-multi-cluster

## Step1. VPC ネットワークの作成

まずは GKE ノードを配置する VPC ネットワークおよび東京リージョンと大阪リージョンにサブネットを作成します。今回の例では GKE ノードからプライベートネットワーク経由で Artifact Registry などの他のマネージドサービスへアクセスできるように限定公開の Google アクセスをオンにしています。

{{< code lang="bash" title="VPCネットワークの作成" >}}
# 環境変数の設定
export NETWORK="matt-vpc"
export SUBNET="matt-private-vm"
export LOCATION_1="asia-northeast1"
export LOCATION_2="asia-northeast2"
export IP_RANGE_1="172.16.0.0/16"
export IP_RANGE_2="172.24.0.0/16"

# VPC ネットワークの作成
gcloud compute networks create ${NETWORK} --subnet-mode=custom

# サブネットの作成 (東京リージョン)
gcloud compute networks subnets create ${SUBNET} \
    --network=${NETWORK} --range=${IP_RANGE_1} --region=${LOCATION_1} \
    --enable-private-ip-google-access

# サブネットの作成 (大阪リージョン)
gcloud compute networks subnets create ${SUBNET} \
    --network=${NETWORK} --range=${IP_RANGE_2} --region=${LOCATION_2} \
    --enable-private-ip-google-access
{{< /code >}}

プライベートネットワーク経由でインターネット上の Docker Hub などへ接続できるよう Cloud NAT も作成しておきます。

{{< code lang="bash" title="Cloud NATの作成" >}}
# 環境変数の設定
export NAT_GATEWAY_1="matt-tokyo-nat"
export NAT_GATEWAY_2="matt-osaka-nat"
export NAT_ROUTER_1="matt-tokyo-router"
export NAT_ROUTER_2="matt-osaka-router"

# Cloud Router の作成 (東京リージョン)
gcloud compute routers create ${NAT_ROUTER_1} \
    --network=${NETWORK} --region=${LOCATION_1}

# Cloud Router の作成 (大阪リージョン)
gcloud compute routers create ${NAT_ROUTER_2} \
    --network=${NETWORK} --region=${LOCATION_2}

# Cloud NAT の作成 (東京リージョン)
gcloud compute routers nats create ${NAT_GATEWAY_1} \
    --router=${NAT_ROUTER_1} \
    --router-region=${LOCATION_1} \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --enable-logging

# Cloud NAT の作成 (大阪リージョン)
gcloud compute routers nats create ${NAT_GATEWAY_2} \
    --router=${NAT_ROUTER_2} \
    --router-region=${LOCATION_2} \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --enable-logging
{{< /code >}}

## Step2. GKE クラスタの作成

次に GKE クラスタを作成していきましょう。 Anthos Service Mesh の導入には次のような前提条件を満たす必要があるため、今回はこちらを満たした上で、セキュリティの観点から限定公開クラスタおよび承認済みネットワークの有効化、安定性の観点から Stable チャンネルを指定しています。

- 4 vCPU 以上を搭載したノード
- 合計 8 vCPU 以上を搭載したノードプール
- GKE Workload Identity の有効化
- GKE リリースチャンネルへの登録 (※1)

※1: Anthos Service Mesh のマネージドコントロールプレーン機能を使う場合のみ

{{< code lang="bash" title="GKEクラスタの作成" >}}
# 環境変数の設定
export PROJECT_ID=`gcloud config list --format "value(core.project)"`
export CLUSTER_1="matt-tokyo-cluster-1"
export CLUSTER_2="matt-osaka-cluster-1"
export MASTER_IP_RANGE_1="192.168.0.0/28"
export MASTER_IP_RANGE_2="192.168.8.0/28"
export CTX_1="gke_${PROJECT_ID}_${LOCATION_1}_${CLUSTER_1}"
export CTX_2="gke_${PROJECT_ID}_${LOCATION_2}_${CLUSTER_2}"

# GKE クラスタ(東京リージョン)の作成
gcloud container clusters create ${CLUSTER_1} \
    --region=${LOCATION_1} \
    --machine-type="e2-standard-4" \
    --num-nodes="1" \
    --enable-autoscaling --min-nodes="1" --max-nodes="3" \
    --enable-private-nodes --master-ipv4-cidr=${MASTER_IP_RANGE_1} \
    --enable-master-global-access \
    --enable-ip-alias --network=${NETWORK} --subnetwork=${SUBNET} \
    --enable-master-authorized-networks \
    --workload-pool="${PROJECT_ID}.svc.id.goog" \
    --release-channel="stable"

# GKE クラスタ(大阪リージョン)の作成
gcloud container clusters create ${CLUSTER_2} \
    --region=${LOCATION_2} \
    --machine-type="e2-standard-4" \
    --num-nodes="1" \
    --enable-autoscaling --min-nodes="1" --max-nodes="3" \
    --enable-private-nodes --master-ipv4-cidr=${MASTER_IP_RANGE_2} \
    --enable-master-global-access \
    --enable-ip-alias --network ${NETWORK} --subnetwork=${SUBNET} \
    --enable-master-authorized-networks \
    --workload-pool="${PROJECT_ID}.svc.id.goog" \
    --release-channel="stable"
{{< /code >}}

前提条件の詳細については次の公式ドキュメントをご参照ください。

https://cloud.google.com/service-mesh/docs/unified-install/prerequisites

## Step3. Anthos Service Mesh のインストール

### 管理ツールのダウンロード

最初に Anthos Service Mesh v1.11 から正式な管理ツールとなった `asmcli` をダウンロードします。

{{< code lang="bash" title="asmcliツールのダウンロード" >}}
curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.11 > asmcli

# 実行権限の付与
chmod +x asmcli
{{< /code >}}

### 東京 GKE クラスタへのインストール

まずは東京リージョンの GKE クラスタに Anthos Service Mesh をインストールしていきましょう。Kubernetes API へ接続できるように GKE コントロールプレーンの承認済みネットワークに Cloud Shell の IP アドレスを登録し、`kubectl` を実行できるようにクラスタ認証情報を取得します。

{{< code lang="bash" title="クラスタ認証情報の取得(東京リージョン)" >}}
# CloudShellの承認済みネットワーク登録
gcloud container clusters update ${CLUSTER_1} \
    --region ${LOCATION_1} \
    --enable-master-authorized-networks \
    --master-authorized-networks \
    "$(dig +short myip.opendns.com @resolver1.opendns.com)/32"

# クラスタ認証情報の取得
gcloud container clusters get-credentials ${CLUSTER_1} \
    --region ${LOCATION_1}
{{< /code >}}

次に `asmcli` を使って Anthos Service Mesh をインストールします。コマンドが完了するまでおおよそ 5 分程度かかりました。

{{< code lang="bash" title="Anthos Service Meshのインストール(東京リージョン)" >}}
./asmcli install \
    --project_id ${PROJECT_ID} \
    --cluster_location ${LOCATION_1} \
    --cluster_name ${CLUSTER_1} \
    --managed \
    --channel "stable" \
    --enable-all \
    --output_dir ${CLUSTER_1}
{{< /code >}}

次のようなメッセージが出力されましたらインストールに成功です。

{{< code lang="txt" title="インストール成功時のメッセージ出力" >}}
asmcli: Successfully installed ASM.
{{< /code >}}

### 大阪 GKE クラスタへのインストール

同様に大阪リージョンの GKE クラスタにもインストールをしましょう。

{{< code lang="bash" title="Anthos Servic eMeshのインストール(大阪リージョン)" >}}
# CloudShellの承認済みネットワーク登録
gcloud container clusters update ${CLUSTER_2} \
    --region ${LOCATION_2} \
    --enable-master-authorized-networks \
    --master-authorized-networks \
    "$(dig +short myip.opendns.com @resolver1.opendns.com)/32"

# クラスタ認証情報の取得
gcloud container clusters get-credentials ${CLUSTER_2} \
    --region ${LOCATION_2}

# Anthos Service Mesh のインストール
./asmcli install \
    --project_id ${PROJECT_ID} \
    --cluster_location ${LOCATION_2} \
    --cluster_name ${CLUSTER_2} \
    --managed \
    --channel "stable" \
    --enable-all \
    --output_dir ${CLUSTER_2}
{{< /code >}}

### ファイアウォールルールの更新 (限定公開クラスタ時のみ)

限定公開クラスタに Anthos Service Mesh をインストールした場合は、コントロールプレーンからのポート 15017 による通信を追加で許可する必要があります。次のコマンド実行してコントロールプレーンからのポート 15017 による通信を許可します。

{{< code lang="bash" title="ファイアウォールルールの更新" >}}
# 既存のファイアウォールルールに 15017/TCP の許可ルールを追加 (東京リージョン)
gcloud compute firewall-rules update \
    `gcloud compute firewall-rules list --filter="name~${CLUSTER_1}-.*-master" --format="value(name)"` \
    --allow tcp:10250,tcp:443,tcp:15017

# 既存のファイアウォールルールに 15017/TCP の許可ルールを追加 (大阪リージョン)
gcloud compute firewall-rules update \
    `gcloud compute firewall-rules list --filter="name~${CLUSTER_2}-.*-master" --format="value(name)"` \
    --allow tcp:10250,tcp:443,tcp:15017
{{< /code >}}

## Step4. マルチクラスタメッシュの設定

### クラスタ間通信の許可

クラスタをまたがってのサービス間通信ができるように次のコマンドを実行してファイアウォールルール `"VPCネットワーク名"-istio-multicluster-pods` を新たに作成します。

{{< code lang="bash" title="クラスタ間通信の許可" >}}
# 環境変数の設定
CLUSTER_1_CIDR=$(gcloud container clusters list \
    --filter="name~${CLUSTER_1}" --format='value(clusterIpv4Cidr)')
CLUSTER_2_CIDR=$(gcloud container clusters list \
    --filter="name~${CLUSTER_2}" --format='value(clusterIpv4Cidr)')
CLUSTER_1_NETTAG=$(gcloud compute instances list \
    --filter="name~${CLUSTER_1::16}" --format='value(tags.items.[0])' | \
    grep ${CLUSTER_1} | sort -u)
CLUSTER_2_NETTAG=$(gcloud compute instances list \
    --filter="name~${CLUSTER_2::16}" --format='value(tags.items.[0])' | \
    grep ${CLUSTER_2} | sort -u)

# クラスタ間通信を許可するファイアウォールルールの作成
gcloud compute firewall-rules create "${NETWORK}-istio-multicluster-pods" \
    --network=${NETWORK} \
    --allow=tcp,udp,icmp,esp,ah,sctp \
    --direction=INGRESS \
    --priority=900 \
    --source-ranges="${CLUSTER_1_CIDR},${CLUSTER_2_CIDR}" \
    --target-tags="${CLUSTER_1_NETTAG},${CLUSTER_2_NETTAG}"
{{< /code >}}

### 承認済みネットワークの追加 (In-cluster かつ承認済みネットワーク有効時のみ)

今回はマネージドコントロールプレーンを利用しているため設定は不要ですが、Anthos Service Mesh を In-cluster で構築した場合は Anthos Service Mesh コントロールプレーンから他 GKE クラスタのコントロールプレーンにアクセスする必要があるため承認済みネットワークを更新します。

{{< code lang="bash" title="承認済みネットワークの追加（In-cluster時のみ）" >}}
# 環境変数の設定
POD_IP_CIDR_1=$(gcloud container clusters describe ${CLUSTER_1} \
    --region ${LOCATION_1} --format "value(ipAllocationPolicy.clusterIpv4CidrBlock)")
POD_IP_CIDR_2=$(gcloud container clusters describe ${CLUSTER_2} \
    --region ${LOCATION_2} --format "value(ipAllocationPolicy.clusterIpv4CidrBlock)")

# 大阪リージョンの Pod アドレス範囲を、東京リージョンの承認済みネットワークに追加
gcloud container clusters update ${CLUSTER_1} \
    --region ${LOCATION_1} \
    --enable-master-authorized-networks \
    --master-authorized-networks \
        "${POD_IP_CIDR_2},$(dig +short myip.opendns.com @resolver1.opendns.com)/32"

# 東京リージョンの Pod アドレス範囲を、大阪リージョンの承認済みネットワークに追加
gcloud container clusters update ${CLUSTER_2} \
    --region ${LOCATION_2} \
    --enable-master-authorized-networks \
    --master-authorized-networks \
        "${POD_IP_CIDR_1},$(dig +short myip.opendns.com @resolver1.opendns.com)/32"
{{< /code >}}

### クラスタ間サービスディスカバリの設定

クラスタ間でサービスの自動検出ができるように `asmcli` を使って設定を行います。

{{< code lang="bash" title="クラスタ間サービスディスカバリの設定" >}}
./asmcli create-mesh ${PROJECT_ID} \
    ${PROJECT_ID}/${LOCATION_1}/${CLUSTER_1} \
    ${PROJECT_ID}/${LOCATION_2}/${CLUSTER_2}
{{< /code >}}

### シークレット情報の更新 (限定公開クラスタ時のみ)

限定公開クラスタで構築した場合は、Anthos Service Mesh コントロールプレーンから他の GKE クラスタコントロールプレーンへプライベートネットワーク経由でアクセスできるようにシークレット情報を書き換えましょう。

{{< code lang="bash" title="シークレット情報の更新(限定公開クラスタ時のみ)" >}}
# 環境変数の設定
CLUSTER_1_PRIV_IP=$(gcloud container clusters describe "${CLUSTER_1}" \
    --region "${LOCATION_1}" --format "value(privateClusterConfig.privateEndpoint)")
CLUSTER_2_PRIV_IP=$(gcloud container clusters describe "${CLUSTER_2}" \
    --region "${LOCATION_2}" --format "value(privateClusterConfig.privateEndpoint)")

# プライベートエンドポイントに書き換えたシークレット情報の作成 (東京リージョン)
./${CLUSTER_1}/istioctl x create-remote-secret \
    --context=${CTX_1} --name=${CLUSTER_1} \
    --server=https://${CLUSTER_1_PRIV_IP} > ${CTX_1}.secret

# プライベートエンドポイントに書き換えたシークレット情報の作成 (大阪リージョン)
./${CLUSTER_1}/istioctl x create-remote-secret \
    --context=${CTX_2} --name=${CLUSTER_2} \
    --server=https://${CLUSTER_2_PRIV_IP} > ${CTX_2}.secret

# シークレット情報の更新 (東京リージョン)
kubectl apply -f ${CTX_2}.secret --context=${CTX_1}

# シークレット情報の更新 (大阪リージョン)
kubectl apply -f ${CTX_1}.secret --context=${CTX_2}
{{< /code >}}

ここまで終わりましたらクラスタ間で Kubernetes サービスがロードバランシングされるようになります。

### クラスタ間ロードバランシングの動作確認

構築はまだ続きますがいったんこの状態で、Anthos Service Mesh をインストールした際に --output_dir で指定したディレクトリへ格納されているサンプルアプリケーションの中から HelloWorld と Sleep というアプリケーションを使用して、クラスタ間で負荷が分散されることを実際に確認していきたいと思います。サンプルアプリケーションの詳細につきましては次の URL をご参照ください。

https://github.com/istio/istio/tree/master/samples

現時点では何もルーティング設定をしていないため、次の図のように 50% ずつトラフィックが振り分けられる状態を確認できるかと思います。

![02-simple-load-balancing-in-mesh.png](images/02-simple-load-balancing-in-mesh.png)

#### サンプルアプリケーションのデプロイ

それではサンプルアプリケーションをデプロイしていきましょう。まずは次のコマンドでサンプルアプリケーション用の Namespace を新たに作成します。

{{< code lang="bash" title="サンプルアプリケーション用Namespaceの作成" >}}
# 環境変数の設定
export SAMPLE_NAMESPACE="sample"

# 両クラスタにサンプルアプリケーション用 Namespace リソースの作成
for CTX in ${CTX_1} ${CTX_2}
do
    kubectl create --context=${CTX} namespace ${SAMPLE_NAMESPACE}
    kubectl label --context=${CTX} namespace ${SAMPLE_NAMESPACE} \
        istio.io/rev=asm-managed-stable --overwrite
done
{{< /code >}}

次に HelloWorld および Sleep アプリケーションをデプロイしましょう。どちらのクラスタ上の Pod に振り分けられたかをわかりやすくするため、東京 GKE クラスタに HelloWorld アプリケーションの v1、大阪 GKE クラスタに v2 をデプロイしています。

{{< code lang="bash" title="サンプルアプリケーションのデプロイ" >}}
# 両クラスタに HelloWorld サービスのデプロイ
for CTX in ${CTX_1} ${CTX_2}
do
    kubectl apply --context=${CTX} -n ${SAMPLE_NAMESPACE} \
        -f ${CLUSTER_1}/istio-1.11.2-asm.17/samples/helloworld/helloworld.yaml \
        -l service="helloworld"
done

# 東京 GKE クラスタに HelloWorld アプリケーションの v1 をデプロイ
kubectl apply --context=${CTX_1} -n ${SAMPLE_NAMESPACE} \
    -f ${CLUSTER_1}/istio-1.11.2-asm.17/samples/helloworld/helloworld.yaml \
    -l version="v1"

# 大阪 GKE クラスタに HelloWorld アプリケーションの v2 をデプロイ
kubectl apply --context=${CTX_2} -n ${SAMPLE_NAMESPACE} \
    -f ${CLUSTER_1}/istio-1.11.2-asm.17/samples/helloworld/helloworld.yaml \
    -l version="v2"

# 両クラスタに Sleep サービス、アプリケーションのデプロイ
for CTX in ${CTX_1} ${CTX_2}
do
    kubectl apply --context=${CTX} -n ${SAMPLE_NAMESPACE} \
        -f ${CLUSTER_1}/istio-1.11.2-asm.17/samples/sleep/sleep.yaml
done
{{< /code >}}

#### サービス間通信の実行

それでは Sleep アプリケーションから HelloWorld アプリケーションへのサービス間通信をしてみましょう。次のコマンドでは各クラスタ上の Sleep アプリケーションからそれぞれ 10 回ずつ HelloWorld サービスへの通信を実施しています。

{{< code lang="bash" title="サービス間通信の実行例" >}}
for CTX in ${CTX_1} ${CTX_2}
do
    echo "--- ${CTX} ---"
    for x in `seq 1 10`
    do
        kubectl exec --context="${CTX}" -n sample -c sleep \
            "$(kubectl get pod --context="${CTX}" -n sample -l \
            app=sleep -o jsonpath='{.items[0].metadata.name}')" \
            -- curl -sS helloworld.${SAMPLE_NAMESPACE}:5000/hello \
                   -w "time_total:%{time_total}\n"
    done
done
{{< /code >}}

次の出力例のように両クラスタから v1 と v2 の Pod へランダムで約 50% ずつトラフィックが振り分けられる状態を確認できるかと思います。また、応答時間については若干ですが東京からのアクセスは東京に配置される v1 の方が良く、大阪については v2 の方が良いことも確認できるかと思います。

{{< code lang="txt" title="メッセージ出力例" >}}
--- gke_${PROJECT_ID}_asia-northeast1_matt-tokyo-cluster-1 ---
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.135174
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.132497
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.175281
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.127934
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.157005
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.141171
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.132390
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.141507
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.129160
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.141614
--- gke_${PROJECT_ID}_asia-northeast2_matt-osaka-cluster-1 ---
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.171108
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.157021
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.137668
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.131152
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.135779
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.141066
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.135495
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.131946
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.135268
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.131380
{{< /code >}}

以上でクラスタ間ロードバランシングの動作確認は完了です。

## Step5. ローカリティロードバランシングの設定

次にもう少し高度なクラスタ間ロードバランシングの設定をしていきましょう。ユースケースはいくつかありますが、今回は次の図のようにサービス間通信は基本的にリージョン内のアプリケーションへルーティングされるようにローカリティを意識した振り分け制御を行っていきたいと思います。

![03-locality-load-balancing-in-mesh.png](images/3-locality-load-balancing-in-mesh.png)

### Istio リソースのデプロイ

それでは設定していきましょう。まずは Istio DestinationRule リソース[^1]の定義ファイルを作成しましょう。例のように DestinationRule リソースにて Outlier Detection(外れ値検知)の設定をすることでローカリティロードバランシングを有効化できます。

[^1]: DestinationRule は転送先サービスのサブセット化や各種トラフィックポリシーを定義する Istio リソース

{{< code lang="yaml" title="helloworld-destinationrule.yaml" >}}
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
{{< /code >}}

次のコマンドで両クラスタに Istio リソースをデプロイしましょう。これで設定は終わりです。

{{< code lang="bash" title="Istioリソースのデプロイ" >}}
# 両クラスタに DestinationRule リソースをデプロイ
for CTX in ${CTX_1} ${CTX_2}
do
    kubectl apply --context=${CTX} -n ${SAMPLE_NAMESPACE} \
        -f helloworld-destinationrule.yaml
done

for CTX in ${CTX_1} ${CTX_2}
do
    kubectl delete --context=${CTX} -n ${SAMPLE_NAMESPACE} \
        -f helloworld-destinationrule.yaml
done
{{< /code >}}

### ローカリティロードバランシングの動作確認

構築はまだ続きますがいったんこの状態で、クラスタ間で負荷が設定どおりの比重で分散されることを実際に確認していきたいと思います。クラスタ間ロードバランシングの動作確認と同様に Sleep アプリケーションから HelloWorld アプリケーションへのサービス間通信をしてみましょう。

{{< code lang="bash" title="サービス間通信の実行例" >}}
for CTX in ${CTX_1} ${CTX_2}
do
    echo "--- ${CTX} ---"
    for x in `seq 1 5`
    do
        kubectl exec --context="${CTX}" -n sample -c sleep \
            "$(kubectl get pod --context="${CTX}" -n sample -l \
            app=sleep -o jsonpath='{.items[0].metadata.name}')" \
            -- curl -sS helloworld.${SAMPLE_NAMESPACE}:5000/hello \
                   -w "time_total:%{time_total}\n"
    done
done
{{< /code >}}

出力例のように東京リージョンの Sleep アプリケーションからのアクセスは v1 のみ、大阪リージョンからのアクセスは v2 のみに振り分けられることが確認できるかと思います。応答時間についてはあまり大きな差は見られないものの、ローカリティを意識しない場合と比較すると若干バラツキが少なくなったように感じます。

{{< code lang="txt" title="メッセージ出力例" >}}
--- gke_${PROJECT_ID}_asia-northeast1_matt-tokyo-cluster-1 ---
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.132195
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.134340
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.127857
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.142889
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.128125
--- gke_${PROJECT_ID}_asia-northeast2_matt-osaka-cluster-1 ---
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.136323
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.131857
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.130870
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.130531
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.131932
{{< /code >}}

以上でローカリティロードバランシング設定の動作確認もできました。

## Step6. Ingress ゲートウェイの設定

さてここからは話題がガラッと変わり、メッシュの外からの通信を受け入れるための Ingress ゲートウェイの設定をしていきたいと思います。今回は次の図のように各クラスタに配置された Ingress ゲートウェイアプリケーションを束ねるようにマルチクラスタ Ingress およびマルチクラスタ Service を配置する構成を作っていきます。

![04-multi-cluster-ingress.png](images/04-multi-cluster-ingress.png)

### マルチクラスタ Ingress 機能の有効化

最初に次のコマンドを実行し、マルチクラスタ Ingress 機能を有効化しましょう。なお、今回はマルチクラスタ Ingress の設定を行うメインの GKE クラスタ(=構成クラスタ)として、GKE クラスタ #1 を登録しています。

{{< code lang="bash" title="マルチクラスタIngress機能の有効化" >}}
gcloud container hub ingress enable \
  --config-membership=${CLUSTER_1}
{{< /code >}}

### Ingress ゲートウェイ定義ファイルの作成

今回は Anthos Service Mesh をインストールした際に `--output_dir` で指定したディレクトリへ Ingress ゲートウェイのサンプル定義ファイルが配置されているのでこちらをベースに作成していきたいと思います。まずはサンプル定義ファイルを複製し、マルチクラスタ Ingress 構成向けに MultiClusterService[^2]、BackendConfig[^3]、MultiClusterIngress[^4] の 3 種類のリソース定義ファイルを追加していきましょう。

[^2]: MultiClusterService は Service リソースを複数のクラスタ上に展開する GKE 独自のカスタムリソース
[^3]: BackendConfig はバックエンドサービスのヘルスチェックを定義する GKE 独自のカスタムリソース
[^4]: MultiClusterIngress はマルチクラスタに対応した Ingress リソースを定義する GKE 独自のカスタムリソース

{{< code lang="bash" title="Ingressゲートウェイ定義ファイルの作成準備" >}}
# サンプル定義ファイルを複製
cp -r ${CLUSTER_1}/samples/gateways/istio-ingressgateway .

# マルチクラスタ Ingress 定義ファイルを格納するディレクトリを作成
mkdir -p istio-ingressgateway/multicluster

# Service リソース定義から MultiClusterService リソース定義ファイルに書き換え
mv istio-ingressgateway/service.yaml istio-ingressgateway/multicluster/multiclusterservice.yaml

# 新たな定義ファイルを 2 種類作成
touch istio-ingressgateway/multicluster/backendconfig.yaml
touch istio-ingressgateway/multicluster/multiclusteringress.yaml
{{< /code >}}

それでは MultiClusterService[^2]、BackendConfig[^3]、MultiClusterIngress[^4] の 3 種類のリソース定義ファイルを編集していきましょう。MultiClusterService は Service リソースをマルチクラスタに対応させたリソースという位置づけのため、基本的に Service リソースの設定値とほぼ変わりません。今回は Ingress をフロントに配置するので LoadBalancer タイプの定義を削除し、デフォルトの Cluster IP に変更しています。

{{< code lang="diff" title="istio-ingressgateway/multicluster/multiclusterservice.yaml（差分）" >}}
- apiVersion: v1
+ apiVersion: networking.gke.io/v1
- kind: Service
+ kind: MultiClusterService
  metadata:
    name: istio-ingressgateway
+   annotations:
+     cloud.google.com/backend-config: '{"default": "ingress-backendconfig"}'
    labels:
      app: istio-ingressgateway
      istio: ingressgateway
  spec:
-   ports:
-   # status-port exposes a /healthz/ready endpoint that can be used with GKE Ingress health checks
-   - name: status-port
-     port: 15021
-     protocol: TCP
-     targetPort: 15021
-   # Any ports exposed in Gateway resources should be exposed here.
-   - name: http2
-     port: 80
-   - name: https
-     port: 443
-   selector:
-     istio: ingressgateway
-     app: istio-ingressgateway
-   type: LoadBalancer
+   template:
+     spec:
+       ports:
+       # status-port exposes a /healthz/ready endpoint that can be used with GKE Ingress health checks
+       - name: status-port
+         port: 15021
+         protocol: TCP
+         targetPort: 15021
+       # Any ports exposed in Gateway resources should be exposed here.
+       - name: http2
+         port: 80
+       - name: https
+         port: 443
+       selector:
+         istio: ingressgateway
+         app: istio-ingressgateway
{{< /code >}}

BackendConfig リソースではバックエンドサービスである Ingress ゲートウェイアプリケーションのヘルスチェックに関する定義を記載します。Ingress ゲートウェイはヘルスチェック用パスとして /healthz/ready:15021 を用意しているため、こちらを設定しましょう。

{{< code lang="diff" title="istio-ingressgateway/multicluster/backendconfig.yaml（差分）" >}}
+ apiVersion: cloud.google.com/v1
+ kind: BackendConfig
+ metadata:
+   name: ingress-backendconfig
+ spec:
+   healthCheck:
+     requestPath: /healthz/ready
+     port: 15021
+     type: HTTP
{{< /code >}}

MultiClusterIngress は Ingress リソースをマルチクラスタに対応させたリソースという位置づけであり、基本的に Ingress リソースを定義するときと設定値はほぼ同じです。

{{< code lang="diff" title="istio-ingressgateway/multicluster/multiclusteringress.yaml（差分）" >}}
+ apiVersion: networking.gke.io/v1beta1
+ kind: MultiClusterIngress
+ metadata:
+   name: istio-ingressgateway
+   labels:
+     app: istio-ingressgateway
+     istio: ingressgateway
+ spec:
+   template:
+     spec:
+       backend:
+         serviceName: istio-ingressgateway
+         servicePort: 80
{{< /code >}}

### Ingress ゲートウェイのデプロイ

まずは Ingress ゲートウェイリソースをデプロイする Namespace を新たに作成します。今回の例では `istio-gateway` という名前の Namespace を作成しています。

{{< code lang="bash" title="Ingress Gateway用のNamespace作成" >}}
# 環境変数の設定
export GATEWAY_NAMESPACE="istio-gateway"

# 両クラスタにサンプルアプリケーション用 Namespace リソースの作成
for CTX in ${CTX_1} ${CTX_2}
do
    kubectl create --context=${CTX} namespace ${GATEWAY_NAMESPACE}
    kubectl label --context=${CTX} namespace ${GATEWAY_NAMESPACE} \
        istio.io/rev=asm-managed-stable --overwrite
done
{{< /code >}}

次のコマンドを実行して Ingress ゲートウェイアプリケーションを両クラスタにデプロイしましょう。

{{< code lang="bash" title="Ingress Gatewayアプリケーションのデプロイ" >}}
for CTX in ${CTX_1} ${CTX_2}
do
    kubectl apply -n ${GATEWAY_NAMESPACE} --context=${CTX} \
        -f istio-ingressgateway
done
{{< /code >}}

最後にマルチクラスタ Ingress リソースを構成クラスタである GKE クラスタ #1 に対してデプロイをしましょう。

{{< code lang="bash" title="Ingress Gatewayアプリケーションのデプロイ" >}}
kubectl apply -n ${GATEWAY_NAMESPACE} --context=${CTX_1} \
    -f istio-ingressgateway/multicluster
{{< /code >}}

以上で Ingress ゲートウェイのデプロイは終わりです。

### Istio リソースのデプロイ

Ingress ゲートウェイを通じてメッシュの外から HelloWorld アプリケーションへ通信ができるように Istio リソースの定義を行っていきたいと思います。まずは Istio Gateway リソース[^5]および Istio VirtualService リソース[^1]の定義ファイルを作成しましょう。例のように Gateway リソースにメッシュ外から受け付けるポートとプロトコルを定義し、VirtualService リソースには Gateway に入ってきた通信のパターンマッチ条件と振り分け先バックエンドの指定をします。

[^5]: Gateway は Ingress/Egress ゲートウェイで受け付けるポート、プロトコルを定義する Istio リソース

{{< code lang="yaml" title="helloworld-gateway.yaml" >}}
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: helloworld-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-gateway
spec:
  hosts:
  - "*"
  gateways:
  - helloworld-gateway
  http:
  - match:
    - uri:
        exact: /hello
    route:
    - destination:
        host: helloworld
        port:
          number: 5000
{{< /code >}}

次のコマンドで両クラスタに Istio リソースをデプロイし、アプリケーションへのインバウンド通信ができるように設定しましょう。

{{< code lang="bash" title="Istioリソースのデプロイ" >}}
for CTX in ${CTX_1} ${CTX_2}
do
    kubectl apply -n ${SAMPLE_NAMESPACE} --context=${CTX} \
        -f helloworld-gateway.yaml
done
{{< /code >}}

以上でメッシュ外からのアプリケーションへのインバウンド通信もできるようになりました。

### インバウンド通信の動作確認

それではメッシュ外からのアプリケーションへのインバウンド通信ができることを確認していきましょう。ローカリティロードバランシングの設定は「Step5. ローカリティロードバランシングの設定」にて実施済みですので次のような挙動となることが想定されます。

![05-locality-load-balancing.png](images/05-locality-load-balancing.png)

#### パブリックエンドポイントの取得

まずは Ingress ゲートウェイの外部 IP アドレスを取得しましょう。

{{< code lang="bash" title="Ingressゲートウェイの外部IPアドレスの取得" >}}
kubectl --context=${CTX_1} \
    -n ${GATEWAY_NAMESPACE} get MultiClusterIngress \
    -o custom-columns=VIP:status.VIP --no-headers
{{< /code >}}

#### インバウンド通信の実行

では東京リージョン上に仮想マシンなどを立てて Ingress ゲートウェイの外部 IP アドレスに対して curl コマンドを実行し、アクセスをしてみましょう。実行例のように v1 に振り分けが 100% されることを確認できるかと思います。

{{< code lang="txt" title="東京からの実行例（ローカリティロードバランシング有効）" >}}
$ for x in `seq 1 10`; do curl http://<Ingress Gateway's External IP>/hello -w "time_total:%{time_total}\n"; done
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.139677
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.139758
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.138518
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.142124
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.137289
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.151358
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.133507
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.139144
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.133560
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.135621
{{< /code >}}

次に大阪リージョン上に仮想マシンなどを立てて Ingress ゲートウェイの外部 IP アドレスに対して curl コマンドを実行し、アクセスをしてみましょう。先ほどとは異なり、実行例のように v2 に振り分けが 100% されることを確認できるかと思います。

{{< code lang="txt" title="大阪からの実行例（ローカリティロードバランシング有効）" >}}
$ for x in `seq 1 10`; do curl http://<Ingress Gateway's External IP>/hello -w "time_total:%{time_total}\n"; done
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.139
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.133
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.134
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.130
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.129
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.141
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.133
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.176
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.137
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.138
{{< /code >}}

#### ご参考、ローカリティロードバランシング無効時の動作

ローカリティロードバランシングを無効(=DestinationRule リソースを削除)にした状態で、各リージョンからアクセスした結果も取得してみました。メッシュ内の通信では差が見えにくかったのですが、インバウンド通信にするとローカリティによるレスポンスの改善がはっきりとわかる結果になりました。

{{< code lang="txt" title="東京からの実行例（ローカリティロードバランシング無効）" >}}
$ for x in `seq 1 10`; do curl http://<Ingress Gateway's External IP>/hello -w "time_total:%{time_total}\n"; done
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.221164
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.142753
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.224596
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.160720
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.188251
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.145336
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.173262
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.138860
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
time_total:0.139937
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
time_total:0.170250
{{< /code >}}

{{< code lang="txt" title="大阪からの実行例（ローカリティロードバランシング無効）" >}}
$ for x in `seq 1 10`; do curl http://<Ingress Gateway's External IP>/hello -w "time_total:%{time_total}\n"; done
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
response_time:0.147
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
response_time:0.225
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
response_time:0.211
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
response_time:0.141
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
response_time:0.134
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
response_time:0.182
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
response_time:0.133
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
response_time:0.134
Hello version: v2, instance: helloworld-v2-54df5f84b-jqpm4
response_time:0.143
Hello version: v1, instance: helloworld-v1-776f57d5f6-6kw7t
response_time:0.187
{{< /code >}}

以上でメッシュの外からアプリケーションへのインバウンド通信に対する動作確認も終了です。お疲れ様でした。

# 終わりに

今回は複数リージョンに展開した複数 GKE クラスタを単一の Anthos Service Mesh 環境に追加し、GKE クラスタ間で負荷分散を行う方法についてご紹介でしたがいかがだったでしょうか。

複数リージョンの GKE クラスタでマルチクラスタメッシュを構築することにより、予期しない大規模災害の発生にも耐えうる高い可用性と回復力の実現、エンドユーザからより近い位置への振り分けによるレイテンシの改善といったことが期待できるようになる見込みです。もしこれから Anthos Service Mesh 環境の利用を検討している方はマルチクラスタメッシュ構成についても検討してみてはいかがでしょうか。

---

- Google Cloud は、Google LLC の商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
