---
title: "サンプルアプリケーションをデプロイする"
description: "サンプル書店アプリケーションをデプロイする"
type: docs
weight: 2
aliases: = ["/docs/install/manual_demo/"]
---

# アプリケーションをデプロイする

このセクションでは、5つの異なるポッドをデプロイし、ポリシーを適用してそれらの間のトラフィックを制御する。

- bookbuyer は、bookstoreにリクエストを行う HTTP クライアントだ。 このトラフィックは**permitted**。
- bookthief は HTTP クライアントであり、bookbuyer と同じように HTTP リクエストを書店に送信する。 このトラフィックは**blocked**される必要がある。
- bookstore は、HTTP リクエストに応答するサーバーだ。また、「bookwarehouse」サービスにリクエストを送信するクライアントでもある。このトラフィックは**permitted**。
- bookwarehouse はサーバーであり、bookstore にのみ応答する必要がある。bookbuyer と bookthief の両方をブロックする必要がある。
- `mysql` は `bookwarehouse` からのみ到達可能な MySQL データベースだ。

[SMI](https://smi-spec.io/) を使用してトラフィックアクセス ポリシーを定義およびデプロイする。これにより、ポッド間で許可およびブロックされたトラフィックの最終的な望ましい状態が得られる。

| from  /   to: | bookbuyer | bookthief | bookstore | bookwarehouse | mysql |
| ------------- | --------- | --------- | --------- | ------------- | ----- |
| bookbuyer     | n/a       | ❌         | ✔         | ❌             | ❌     |
| bookthief     | ❌         | n/a       | ❌         | ❌             | ❌     |
| bookstore     | ❌         | ❌         | n/a       | ✔             | ❌     |
| bookwarehouse | ❌         | ❌         | ❌         | n/a           | ✔     |
| mysql         | ❌         | ❌         | ❌         | ❌             | n/a   |


SMI Traffic Split を使用してトラフィックを分割する方法を示すために、追加のアプリケーションをデプロイする。

- `bookstore-v2` - これはデプロイした最初の書店と同じコンテナーだが、このデモでは、アップグレードする必要があるアプリの新しいバージョンであると想定する。

bookbuyer、bookthief、bookstore、および bookwarehouse ポッドは、同じ名前の別の Kubernetes Namespace に配置される。`mysql` は `bookwarehouse` 名前空間にある。サービス メッシュ内の新しい各ポッドには、Pipy サイドカーコンテナーが挿入される。

### 名前空間を作成する

```bash
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
```

### 新しい名前空間を osm-edge コントロールプレーンに追加する

```bash
osm namespace add bookstore bookbuyer bookthief bookwarehouse
```

現在、4つの名前空間のそれぞれに openservicemesh.io/monitored-by: osmというラベルが付けられ、openservicemesh.io/sidecar-injection: enabled という注釈も付けられている。osm-edgeコントローラは、これらの名前空間のラベルとアノテーションに注意し、すべての**new**ポッドにPipyのサイドカーを注入し始める。

### ポッド、サービス、サービスアカウントを作成する。

「bookbuyer」サービスアカウントとデプロイメントを作成する。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookbuyer.yaml
```

「bookthief」サービスアカウントとデプロイメントを作成する。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookthief.yaml
```

「bookstore」サービスアカウント、サービス、デプロイメントを作成する。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookstore.yaml
```

「bookwarehouse」サービスアカウント、サービス、デプロイメントを作成する。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookwarehouse.yaml
```

「mysql」サービスアカウント、サービス、およびステートフルセットを作成する。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/mysql.yaml
```

### チェックポイント: 何がインストールされたか?

「bookbuyer」、「bookthief」、「bookstore」、「bookwarehouse」のそれぞれの Kubernetes デプロイメント とポッド、および「mysql」の ステートフルセット。また、「bookstore」、「bookwarehouse」、及び「mysql」の Kubernetes サービスとエンドポイント。

クラスターでこれらのリソースを表示するには、次のコマンドを実行する。

```bash
kubectl get pods,deployments,serviceaccounts -n bookbuyer
kubectl get pods,deployments,serviceaccounts -n bookthief

kubectl get pods,deployments,serviceaccounts,services,endpoints -n bookstore
kubectl get pods,deployments,serviceaccounts,services,endpoints -n bookwarehouse
```

さらに、アプリケーションごとに [Kubernetes Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)も作成された。サービスアカウントは、サービス間のアクセス制御ポリシーを作成するためにデモで後で使用されるアプリケーションの ID として機能する。
### アプリケーション UI を表示する

次の手順でクライアントポートフォワーディングを設定し、Kubernetes クラスター内のアプリケーションにアクセスする。元の端末を使用してコマンドの発行を続行しながら、ポート転送セッションを維持するためにポート転送スクリプトを実行するための新しい端末セッションを開始することをお勧めする。port-forward-all.sh スクリプトは、スクリプトの実行に必要な環境変数の .env ファイルを探す。.env は、以前に作成された名前空間を対象とする必要な変数を作成する。参照 .env.example ファイルを使用してから、ポート転送スクリプトを実行する。 

新しいターミナルセッションで、次のコマンドを実行して、プロジェクトディレクトリのルート ([upstream osm-edge](https://github.com/flomesh-io/osm-edge)のローカルクローン) から Kubernetes クラスターへのポート転送を有効にする。

```bash
cp .env.example .env
./scripts/port-forward-all.sh
```

_注意: デフォルトのポートをオーバーライドするには、BOOKBUYER_LOCAL_PORT、BOOKSTORE_LOCAL_PORT、BOOKSTOREv1_LOCAL_PORT、BOOKSTOREv2_LOCAL_PORT、BOOKTHIEF_LOCAL_PORT 変数の割り当てをポート転送スクリプトにプレフィックスとして付ける。例えば:_

```bash
BOOKBUYER_LOCAL_PORT=7070 BOOKSTOREv1_LOCAL_PORT=7071 BOOKSTOREv2_LOCAL_PORT=7072 BOOKTHIEF_LOCAL_PORT=7073 BOOKSTORE_LOCAL_PORT=7074 ./scripts/port-forward-all.sh
```

ブラウザーで、次の URL を開く。

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**
- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
  - _注意: このページは、現時点ではデモでは利用できない。これは、SMIトラフィック分割構成のセットアップ中に使用可能になる_

4 つすべてが同時に見えるようにウィンドウを配置する。ウェブ ページの上部にあるヘッダーは、アプリケーションとバージョンを示す。

## 次のステップ

サンプル アプリケーションが実行されているので、アプリケーション間の[configure traffic policies](/docs/getting_started/traffic_policies/)。
