---
title: 「トレース」
description: 「イェーガーとの追跡」
type: docs
weight: 4
---

# トレース
osm-edge では、トレース用に Jaeger をオプションでデプロイできます。 同様に、インストール時 (`values.yaml` の `tracing` セクション) または実行時に `osm-mesh-config` カスタム リソースを編集して、トレースを有効にしてカスタマイズすることができます。 BYO シナリオをサポートするために、いつでもトレースを有効化、無効化、および構成できます。

トレースを有効にして osm-edge がデプロイされると、osm-edge コントロール プレーンは [user-provided tracing information](#tracing-values) を使用して、適切な場合に適切な場所でトレースを送信するように Pipy に指示します。 ユーザー提供の値なしでトレースが有効になっている場合は、`values.yaml` のデフォルトが使用されます。 `tracing-address` の値は、osm-edge によって注入されたすべての Pipy にトレース情報を送信する FQDN を伝えます。

osm-edge は、Zipkin プロトコルを使用するアプリケーションでのトレースをサポートします。

## イェーガー

[Jaeger](https://www.jaegertracing.io/) は、分散システムの監視とトラブルシューティングに使用されるオープン ソースの分散トレーシング システムです。 これにより、セットアップ全体できめ細かいメトリクスと分散トレース情報を取得できるため、どのマイクロサービスが通信しているか、リクエストがどこに行き、どれくらいの時間がかかっているかを観察できます。 これを使用して、特定のリクエストとレスポンスを調べて、それらがいつどのように発生するかを確認できます。

トレースが有効になっている場合、Jaeger はメッシュ内の Pipy からスパンを受信でき、ポート転送を介して Jaeger の UI で表示およびクエリできます。

osm-edge CLI は、osm-edge のインストールで Jaeger インスタンスをデプロイする機能を提供しますが、独自の管理された Jaeger を持ち込み、後でそれを指すように osm-edge のトレースを構成することもサポートされています。

### Jaeger の自動プロビジョニング

デフォルトでは、Jaeger のデプロイメントとトレースは全体として無効になっています。
Jaeger インスタンスは、インストール時に `--set=osm.deployJaeger=true` osm-edge CLI フラグを使用して自動的にデプロイできます。 これにより、メッシュ名前空間に Jaeger ポッドがプロビジョニングされます。

さらに、プロキシでのトレースを有効にするように osm-edge に指示する必要があります。 これは、MeshConfig の「tracing」セクションを介して行われます。

次のコマンドは、osm-edge インストール中に Jaeger の新しくデプロイされたインスタンスのアドレスに従って、Jaeger をデプロイし、トレース パラメータを構成します。
```bash
osm install --set=osm.deployJaeger=true,osm.tracing.enable=true
```

このデフォルトのブリングアップは、Jaeger UI、コレクター、クエリ、およびエージェントを起動する [All-in-one Jaeger executable](https://www.jaegertracing.io/docs/1.22/getting-started/#all-in-one)を使用します。

### BYO（持ち込み）
このセクションでは、すでに実行中の Jaeger のインスタンスを osm-edge コントロール プレーンと統合できるようにするために必要な追加の手順について説明します。
> 注: このガイドでは Jaeger に特化した手順の概要を説明していますが、独自のトレース アプリケーション インスタンスを適切な値で使用することもできます。 osm-edge は、Zipkin プロトコルを使用するアプリケーションでのトレースをサポートします

#### 前提条件
* 実行中の Jaeger インスタンス
    *  [Getting started with Jaeger](https://www.jaegertracing.io/docs/1.22/getting-started/)にサンプルアプリをデモとして同梱
#### トレース値

以下のセクションでは、既に osm-edge をインストールしているか、osm-edge インストール中にトレースと Jaeger をデプロイしているかに応じて、必要な更新を行う方法について概説します。 いずれの場合も、「values.yaml」内の次の「tracing」値は、Jaeger インスタンスを指すように更新されています。
1. `enable`: `true` に設定して、特定のアドレス (クラスター) にトレース データを送信するように Pipy 接続マネージャーに指示します。
1. `address`: Jaeger インスタンスの宛先クラスターに設定
1. `port`: 使用するリスナーの宛先ポートに設定します
1. `endpoint`: スパンが送信される宛先の API またはコレクター エンドポイントに設定します。


#### a) osm-edge コントロール プレーンのインストール後にトレースを有効にする

すでに osm-edge を実行している場合は、次を使用して osm-edge MeshConfig で `tracing` 値を更新する必要があります。

```bash
# Tracing configuration with sample values
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"observability":{"tracing":{"enable":true,"address": "jaeger.osm-system.svc.cluster.local","port":9411,"endpoint":"/api/v2/spans"}}}}'  --type=merge
```

「osm-mesh-config」リソースを調べることで、これらの変更がデプロイされたことを確認できます。
```bash
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.observability.tracing}{"\n"}'
```

#### b) osm-edge コントロール プレーンのインストール時にトレースを有効にする

osm-edge のインストール中に Jaeger の独自のインスタンスをデプロイするには、以下に示すように `--set` フラグを使用して値を更新します。

```bash
osm install --set osm.tracing.enable=true,osm.tracing.address=<tracing server hostname>,osm.tracing.port=<tracing server port>,osm.tracing.endpoint=<tracing server endpoint>
```

## ポート転送を使用して Jaeger UI を表示する
Jaeger の UI はポート 16686 で実行されています。Web UI を表示するには、「kubectl port-forward」を使用できます。

```bash
OSM_POD=$(kubectl get pods -n "$K8S_NAMESPACE" --no-headers  --selector app=jaeger | awk 'NR==1{print $1}')

kubectl port-forward -n "$K8S_NAMESPACE" "$OSM_POD"  16686:16686
```
Web ブラウザーで「http://localhost:16686/」 に移動して、UIを表示します。


## Jaeger を使用したトレースの例
このセクションでは、単純な Jaeger インスタンスを作成し、osm-edge で Jaeger を使用してトレースを有効にするプロセスについて説明します。

1. Jaeger をデプロイして [osm-edge Demo](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/README.md) を実行します。 次の 2 つのオプションがあります。
    - Jaeger の自動プロビジョニングの場合は、`.env` ファイルで `DEPLOY_JAEGER` を true に設定するだけです
     - 持ち込みの場合、以下のコマンドを使用して、サンプル インスタンス [provided by Jaeger](https://www.jaegertracing.io/docs/1.22/getting-started/#all-in-one) をデプロイできます。 別の名前空間で Jaeger を立ち上げたい場合は、必ず以下で更新してください。

        Jaeger サービスを作成します。
        ```yaml
        kubectl apply -f - <<EOF
        ---
        kind: Service
        apiVersion: v1
        metadata:
          name: jaeger
          namespace: osm-system
          labels:
            app: jaeger
        spec:
          selector:
            app: jaeger
          ports:
          - protocol: TCP
            # Service port and target port are the same
            port: 9411
          type: ClusterIP
        EOF
        ```

        Jaeger デプロイメントを作成します。
        ```yaml
        kubectl apply -f - <<EOF
        ---
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: jaeger
          namespace: osm-system
          labels:
            app: jaeger
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: jaeger
          template:
            metadata:
              labels:
                app: jaeger
            spec:
              containers:
              - name: jaeger
                image: jaegertracing/all-in-one
                args:
                  - --collector.zipkin.host-port=9411
                imagePullPolicy: IfNotPresent
                ports:
                - containerPort: 9411
                resources:
                  limits:
                    cpu: 500m
                    memory: 512M
                  requests:
                    cpu: 100m
                    memory: 256M
        EOF
        ```

1. トレースを有効にして、適切な値を渡します。 Jaeger を別の名前空間にインストールした場合は、以下の「osm-system」を置き換えます。

    ```bash
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"observability":{"tracing":{"enable":true,"address": "jaeger.osm-system.svc.cluster.local","port":9411,"endpoint":"/api/v2/spans"}}}}'  --type=merge
    ```

1. ポート転送を使用して Web UI を表示するには、[above](#view-the-jaeger-ui-with-port-forwarding) の手順を参照してください。
1. ブラウザに「Service」ドロップダウンが表示され、bookstore デモによってデプロイされたさまざまなアプリケーションから選択できます。

    a) サービスを選択して、そこからすべてのスパンを表示します。 たとえば、1 時間の「Lookback」で「bookbuyer」を選択すると、「bookstore-v1」および「bookstore-v2」とのやり取りを時間順に並べて表示できます。
    <p align="center">
        <img src="/docs/images/jaeger-search-traces.png" width="100%"/>
    </p>
    <center><i>Jaeger UI search for bookbuyer traces</i></center><br>

    b) 項目をクリックすると、詳細が表示されます

    c) 複数の項目を選択してトレースを比較します。 たとえば、特定の時点での「bookbuyer」と「bookstore-v1」および「bookstore-v2」とのやり取りを比較できます。
    <p align="center">
        <img src="/docs/images/jaeger-compare-traces.png" width="100%"/>
    </p>
    <center><i>bookbuyer interactions with bookstore-v1 and bookestore-v2</i></center><br>

    d) [システム アーキテクチャ] タブをクリックして、さまざまなアプリケーションがどのように相互作用/通信しているかを示すグラフを表示します。 これにより、アプリケーション間でトラフィックがどのように流れているかがわかります。
    <p align="center">
        <img src="/docs/images/jaeger-system-architecture.png" width="40%"/>
    </p>
    <center><i>Directed acyclic graph of bookstore demo application interactions</i></center><br>

Jaeger UI に本屋のデモ アプリケーションが表示されない場合は、`bookbuyer` ログを追跡して、アプリケーションが正常に対話していることを確認してください。

```bash
POD="$(kubectl get pods -n "$BOOKBUYER_NAMESPACE" --show-labels --selector app=bookbuyer --no-headers | grep -v 'Terminating' | awk '{print $1}' | head -n1)"

kubectl logs "${POD}" -n "$BOOKBUYER_NAMESPACE" -c bookbuyer --tail=100 -f
```

次のことを期待します。
```bash
"MAESTRO! THIS TEST SUCCEEDED!"
```
これは、Jaeger またはトレース構成が原因で問題が発生していないことを示しています。

## アプリケーションに Jaeger トレースを統合する

Jaeger のトレースは簡単にはできません。 Jaeger が要求をトレースに自動的に関連付けるためには、トレース情報を正しく公開することがアプリケーションの責任です。

Open Service Mesh のサイドカー プロキシ構成では、現在、Zipkin が HTTP トレーサとして使用されています。 したがって、アプリケーションは Zipkin がサポートするヘッダーを利用して、トレース情報を提供できます。 トレースの最初のリクエストで、Zipkin プラグインは必要な HTTP ヘッダーを生成します。 後続のリクエストを現在のトレースに追加する必要がある場合、アプリケーションは以下のヘッダーを伝播する必要があります。

* `x-request-id`
* `x-b3-traceid`
* `x-b3-spanid`
* `x-b3-parentspanid`

## トレース/Jaeger のトラブルシューティング

トレースが期待どおりに機能しない場合。

### 1. トレースが有効になっていることを確認する
「tracing」構成の「enable」キーが「true」に設定されていることを確認します。
```bash
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.observability.tracing.enable}{"\n"}'
true
```

### 2. 設定されているトレース値が期待どおりであることを確認します
トレースが有効になっている場合、`osm-mesh-config` リソースでトレースに使用されている特定の `address`、`port`、および `endpoint` を確認できます。
```bash
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.observability.tracing}{"\n"}'
```
Pipy が使用する FQDN を指していることを確認するには、`address` キーの値を確認します。

### 3. 使用されているトレース値が期待どおりであることを確認します
1 レベル深く掘り下げるために、MeshConfig によって設定された値が正しく使用されているかどうかを確認することもできます。 以下のコマンドを使用して、問題のポッドの構成ダンプを取得し、出力をファイルに保存します。
```bash
osm proxy get config_dump -n <pod-namespace> <pod-name> > <file-name>
```
お気に入りのテキスト エディターでファイルを開き、「pipy-tracing-cluster」を検索します。 使用中のトレース値を確認できるはずです。 bookbuyer ポッドの出力例:
```console
"name": "pipy-tracing-cluster",
      "type": "LOGICAL_DNS",
      "connect_timeout": "1s",
      "alt_stat_name": "pipy-tracing-cluster",
      "load_assignment": {
       "cluster_name": "pipy-tracing-cluster",
       "endpoints": [
        {
         "lb_endpoints": [
          {
           "endpoint": {
            "address": {
             "socket_address": {
              "address": "jaeger.osm-system.svc.cluster.local",
              "port_value": 9411
        [...]
```

### 4. Jaeger が自動的にデプロイされた状態で osm-edge コントローラーがインストールされたことを確認します [オプション]
自動ブリングアップを使用した場合は、さらに Jaeger サービスと Jaeger デプロイメントを確認できます。
```bash
# Assuming osm-edge is installed in the osm-system namespace:
kubectl get services -n osm-system -l app=jaeger

NAME     TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
jaeger   ClusterIP   10.99.2.87   <none>        9411/TCP   27m
```

```bash
# Assuming osm-edge is installed in the osm-system namespace:
kubectl get deployments -n osm-system -l app=jaeger

NAME     READY   UP-TO-DATE   AVAILABLE   AGE
jaeger   1/1     1            1           27m
```

### 5. Jaeger ポッドの準備状況、応答性、正常性を確認する
デプロイした名前空間で Jaeger ポッドが実行されているかどうかを確認します。
> 以下のコマンドは、osm-edge の Jaeger の自動展開に固有のものです。 必要に応じて、独自のトレース インスタンスの名前空間とラベルの値を置き換えます。
```bash
kubectl get pods -n osm-system -l app=jaeger

NAME                     READY   STATUS    RESTARTS   AGE
jaeger-8ddcc47d9-q7tgg   1/1     Running   5          27m
```

Jaeger インスタンスに関する情報を取得するには、「kubectl describe pod」を使用して、出力の「Events」を確認します。
```bash
kubectl describe pod -n osm-system -l app=jaeger
```

### 外部リソース
* [Jaeger Troubleshooting docs](https://www.jaegertracing.io/docs/1.22/troubleshooting/)
