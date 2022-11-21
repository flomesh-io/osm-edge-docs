---
title: 「クイックスタート」
description: "osm-edge を 5 分で試す"
type: docs
Weight: 2
---

# osm-edge クイック スタート ガイド

このガイドでは、osm-edge をダウンロード、インストール、実行し、デモ アプリケーションを展開し、リンク暗号化、アクセス制御、トラフィック分割などの SMI 標準機能を 5 分未満で完了する方法を示します。 このデモは、x86 アーキテクチャで Ubuntu 21 を実行していて、k3s バージョン 'V1.23.8 + K3S1' を実行していることを前提としています。 バージョンとプラットフォームのサポートの詳細については、完全な [Beginner's Guide](docs/getting_started/). を参照してください。

## 前提条件

Kubernetes クラスターを実行しています。 お持ちでない場合は、以下のスクリプトを使用して k3s をインストールできます。

```bash
export INSTALL_K3S_VERSION=v1.23.8+k3s1
curl -sfL https://get.k3s.io | sh -s - --disable traefik --write-kubeconfig-mode 644 --write-kubeconfig ~/.kube/config
```

>osm-edge がサポートする Kubernetes の最小バージョンは {{< param min_k8s_version >}} です

## osm-edge CLI をダウンロードしてインストールする

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release={{< param osm_edge_version >}}
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

## Kubernetes クラスタに osm-edge をインストールする

> 以下のコマンドは、[Prometheus](https://github.com/prometheus/prometheus)、[Grafana](https://github.com/grafana/grafana)、および [Jaeger](https://github.com/jaegertracing/jaeger)をインストールして有効にします。

```bash
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.deployPrometheus=true \
    --set=osm.deployGrafana=true \
    --set=osm.deployJaeger=true \
    --set=osm.tracing.enable=true
```
## アプリケーションのデプロイ

このセクションでは、5 つの異なる Pod をデプロイし、ポリシーを適用してそれらの間のトラフィックを制御します。

- 「bookbuyer」は、「bookstore」にリクエストを送信する HTTP クライアントです。 このトラフィックは **許可されています**。
- 「bookthief」は HTTP クライアントであり、「bookbuyer」と同様に「bookstore」に対して HTTP リクエストを行います。 このトラフィックは**ブロック**する必要があります。
- `bookstore` は、HTTP リクエストに応答するサーバーです。 「bookwarehouse」サービスにリクエストを行うクライアントでもあります。 このトラフィックは **許可されています**。
- `bookwarehouse` はサーバーであり、`bookstore` にのみ応答する必要があります。 「bookbuyer」と「bookthief」の両方をブロックする必要があります。
- `mysql` は、`bookwarehouse` によってのみ到達可能な MySQL データベースです。

以下のスクリプトを使用してインストールします。

```bash
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
osm namespace add bookstore bookbuyer bookthief bookwarehouse
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookbuyer.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookthief.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookstore.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookwarehouse.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/mysql.yaml
```

各サービスの GUI ポートを公開して、ブラウザーでデモ アプリケーションのこれらのポートにアクセスできるようにします。

```bash
git clone https://github.com/flomesh-io/osm-edge.git -b {{< param osm_branch >}}
cd osm-edge
cp .env.example .env
./scripts/port-forward-all.sh #可以忽略错误信息
```

ブラウザーで、次の URL を開きます。

_注: ホストからアクセスする必要がある場合は、「localhost」を仮想マシンの IP アドレスに置き換える必要があります。 または、ホストで「port-forward-all.sh」スクリプトを実行します。 _

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**
- [http://localhost:8084](http://localhost:8084) - **bookstore**


## アクセス制御

上記のコマンドで osm-edge をインストールすると、すべてのサービスがアクセス制御なし (寛容なトラフィック ポリシー モード) になるか、すべてのアクセスが許可されます。 アクセス制御がない場合の状況は、ブラウザのサービスごとの書籍数の増加を見るとわかります。

「bookbuyer」UIと「bookthief」UI のカウントは、それぞれ購入した本の数と盗まれた本の数に対応していますが、「bookstore-v1」では、これらは増加しているはずです。

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**

「bookstore」UI での本の販売数も増加しているはずです。

- [http://localhost:8084](http://localhost:8084) - **bookstore**

以下は、permissive トラフィック ポリシー モードを無効にして「bookstore」サービスへのアクセスを拒否する方法を示しています。

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

カウントが増加していないことがわかります。

以下のコマンドを実行して、「bookbuyer」 権限が 「bookstore」 にアクセスできるようにします。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/access/traffic-access-v1.yaml
```

ここで「bookbuyer」と「bookstore」の UI に戻り、「bookthief」の UI のカウントが停止したままで、カウントが再び増加していることを確認します。

通常の購入は影響を受けませんが、アクセス制御により、「bookthief」が「bookstore」から本を盗むのを防ぐことに成功しました。

## 可観測性

### 指標

以下のコマンドを使用して、ネームスペース メトリックの生成とキャプチャを有効にします。そうしないと、Pod によって生成されたメトリックが収集されません。

```shell
osm metrics enable --namespace "bookstore,bookbuyer,bookthief,bookwarehouse"
```

ポート転送スクリプトを実行した後、ブラウザで URL `http://localhost:3000` を開いて Grafan コンソールにアクセスします。 ダッシュボードのデフォルトのユーザー名とパスワードは「admin」、「admin」です。

osm-edge には、コントロール プレーンとデータ プレーンでメトリックを視覚化するためのダッシュボードがいくつか組み込まれています。 たとえば、次の図は、他の「サービス」にアクセスする「bookthief」サービスのポッド「http://localhost:3000」のメトリックを示しています。

![image](https://user-images.githubusercontent.com/2224492/180593501-d73dbf11-40a8-4fe9-9422-ea931da2927f.png)

次の図は、「デプロイ」の粒度で他の「サービス」にアクセスする「bookthief」のメトリクスを示しています。 前の図との違いは、「bookthief」に複数のレプリカがある場合、すべてのレプリカの集計データが次のように表示されることです。

![image](https://user-images.githubusercontent.com/2224492/180593509-9a852bf1-e7e7-4534-9c57-06cf1c890ee3.png)

osm-edge コンポーネントの次のメトリックと、メッシュ ベース情報がここに表示されます。

![image](https://user-images.githubusercontent.com/2224492/180593512-0ac33a0e-2b7a-4e66-b499-f196b5dd729b.png)

### トレース

Jaeger のダッシュボードには、ブラウザーに「http://localhost:16686/search」と入力してアクセスできます。

![image](https://user-images.githubusercontent.com/2224492/180593520-64b0d2d1-1346-47ac-aab8-a9eaae9f8950.png)

ダッシュボードでは、サービス関連のトレース情報を検索できます: !

![image](https://user-images.githubusercontent.com/2224492/180593525-3bc844c4-f950-48f6-9d72-ff98dc82aa2c.png)

サービス トポロジ図を表示します。

![image](https://user-images.githubusercontent.com/2224492/180593530-8d0ed18f-0cac-495f-985f-04feb863ec6d.png)

### ロギング

osm-edge コントロール プレーンは、サービス メッシュ管理のために診断ログを標準出力に出力します。ログ情報の出力は、ログのレベルを調整することで制御できます。 標準出力に出力されたログは、ログ収集ツールで集計して保存できます。

## サービス メッシュのアンインストール
osm-edge のクイック エクスペリエンスを完了した後、osm-edge に関連付けられているすべてのリソースをアンインストールするには、これらのサンプル アプリケーションと関連する SMI リソースを削除し、osm-edge コントロール プレーンとクラスター全体の osm-edge リソースをアンインストールする必要があります。

サンプル アプリケーションを削除するには。

```shell
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

コントロール プレーンをアンインストールします。

```shell
osm uninstall mesh
```