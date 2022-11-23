---
title: 「指標」
description: 「プロキシおよび osm-edge コントロール プレーンの Prometheus メトリクス」
type: docs
weight: 1
---

# 指標
osm-edge は、メッシュおよび osm-edge コントロール プレーン内のすべてのトラフィックに関連する詳細なメトリックを生成します。 これらのメトリクスは、メッシュ内のアプリケーションの動作とメッシュ自体に関する洞察を提供し、ユーザーがアプリケーションのトラブルシューティング、保守、分析を行うのに役立ちます。

osm-edge は、サイドカー プロキシ (Pipy) からメトリクスを直接収集します。 これらのメトリックを使用すると、ユーザーはトラフィックの全体量、トラフィック内のエラー、およびリクエストの応答時間に関する情報を取得できます。

さらに、osm-edge は、コントロール プレーン コンポーネントのメトリックを生成します。 これらのメトリックを使用して、サービス メッシュの動作と正常性を監視できます。

osm-edge は [Prometheus][1] を使用して、メッシュで実行されているすべてのアプリケーションの一貫したトラフィック メトリックと統計を収集して保存します。 Prometheus は、オープンソースの監視およびアラート ツールキットであり、Kubernetes および Service Mesh 環境で一般的に使用されます (ただし、これらに限定されません)。

メッシュの一部である各アプリケーションは、Prometheus 形式でメトリクス (プロキシ メトリクス) を公開する Pipy サイドカーを含む Pod で実行されます。 さらに、メッシュの一部であり、メトリクスが有効になっている名前空間にあるすべての Pod には、Prometheus アノテーションがあり、Prometheus サーバーがアプリケーションを動的にスクレイピングできるようにします。 このメカニズムにより、ポッドがメッシュに追加されるたびに、メトリックのスクレイピングが自動的に有効になります。

osm-edge メトリクスは、オープンソースの視覚化および分析ソフトウェアである [Grafana][8] で表示できます。 これにより、メトリックのクエリ、視覚化、アラート、および探索が可能になります。

Grafana は Prometheus をバックエンドの時系列データベースとして使用します。 Grafana と Prometheus が osm-edge インストールによってデプロイされるように選択された場合、それらが相互作用するために必要なルールがデプロイ時に設定されます。 逆に、「Bring-Your-Own」または「BYO」モデル (以下で詳しく説明) では、これらのコンポーネントのインストールはユーザーが行います。

## メトリクス コンポーネントのインストール

osm-edge は、インストール時に Prometheus および Grafana インスタンスをプロビジョニングするか、osm-edge を既存の Prometheus および/または Grafana に接続できます。
実例。 後者のパターンを「Bring-Your-Own」または「BYO」と呼びます。 以下のセクションでは、osm-edge を許可してメトリクスを構成する方法について説明します
メトリクス コンポーネントを自動的にプロビジョニングし、BYO メソッドを使用します。

### 自動プロビジョニング

デフォルトでは、Prometheus と Grafana の両方が無効になっています。

ただし、`--set=osm.deployPrometheus=true` フラグで設定すると、osm-edge インストールは Prometheus インスタンスをデプロイして、サイドカーのメトリクス エンドポイントを取得します。 ユーザーが設定したメトリクス スクレイピング構成に基づいて、osm-edge はメッシュのポッド部分に必要なメトリクス アノテーションを付けて、Prometheus がポッドに到達してスクレイピングし、関連するメトリクスを収集できるようにします。 [scraping configuration file](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/templates/prometheus-configmap.yaml) は、デフォルトの Prometheus の動作とセットを定義します osm-edge によって収集されたメトリクス。

メトリクスの視覚化のために Grafana をインストールするには、`--set=osm.deployGrafana=true` フラグを `osm install` コマンドに渡します。 osm-edge は、[osm-edge Grafana dashboards](#osm-grafana-dashboards) に記載されている事前構成済みのダッシュボードを提供します。

```bash
 osm install --set=osm.deployPrometheus=true \
             --set=osm.deployGrafana=true
```

> 注: osm-edge によって自動的にデプロイされる Prometheus および Grafana インスタンスは、高可用性、永続ストレージ、またはロックダウン セキュリティを含まないシンプルな構成です。 本番グレードのインスタンスが必要な場合は、それらを事前にプロビジョニングし、このページの BYO の手順に従ってそれらを osm-edge と統合してください。

### 持込可

#### プロメテウス

次のセクションでは、すでに実行中の Prometheus インスタンスが osm-edge メッシュのエンドポイントをポーリングできるようにするために必要な追加の手順について説明します。

##### BYO Prometheus の前提条件のリスト

- メッシュの _outside_ でアクセス可能な Prometheus インスタンスを既に実行しています。
- メトリクス スタックなしでデプロイされた、実行中の osm-edge コントロール プレーン インスタンス。
- Grafana が Prometheus に到達すること、Prometheus または Grafana Web ポートを公開または転送すること、および Kubernetes API サービスに到達するように Prometheus を構成することは、処理されるか、これらの手順の範囲外であると想定します。

##### 構成

- Prometheus インスタンスに、ポッドと Kubernetes API の両方に到達できる適切な RBAC ルールがあることを確認してください。これは、さまざまなデプロイの特定の要件と状況に依存する場合があります。
```yaml
- apiGroups: [""]
   resources: ["nodes", "nodes/proxy",  "nodes/metrics", "services", "endpoints", "pods", "ingresses", "configmaps"]
   verbs: ["list", "get", "watch"]
 - apiGroups: ["extensions"]
   resources: ["ingresses", "ingresses/status"]
   verbs: ["list", "get", "watch"]
 - nonResourceURLs: ["/metrics"]
   verbs: ["get"]
```

- 必要に応じて、Prometheus サービス定義を使用して、Prometheus が自身をスクレイピングできるようにします。
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "<API port for prometheus>" # Depends on deployment - osm-edge automatic deployment uses 7070 by default, controlled by `values.yaml`
```

- Pod/Pipy エンドポイントに到達するように Prometheus の configmap を修正します。 osm-edge は自動的にポート アノテーションを Pod に追加し、Prometheus が到達できるようにリスナー構成を Pod にプッシュします。
```yaml
- job_name: 'kubernetes-pods'
   kubernetes_sd_configs:
   - role: pod
   relabel_configs:
   - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
   - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
   - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
   - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: source_namespace
   - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: source_pod_name
   - regex: '(__meta_kubernetes_pod_label_app)'
      action: labelmap
      replacement: source_service
   - regex: '(__meta_kubernetes_pod_label_osm_sidecar_uid|__meta_kubernetes_pod_label_pod_template_hash|__meta_kubernetes_pod_label_version)'
      action: drop
   - source_labels: [__meta_kubernetes_pod_controller_kind]
      action: replace
      target_label: source_workload_kind
   - source_labels: [__meta_kubernetes_pod_controller_name]
      action: replace
      target_label: source_workload_name
   - source_labels: [__meta_kubernetes_pod_controller_kind]
      action: replace
      regex: ^ReplicaSet$
      target_label: source_workload_kind
      replacement: Deployment
   - source_labels:
      - __meta_kubernetes_pod_controller_kind
      - __meta_kubernetes_pod_controller_name
      action: replace
      regex: ^ReplicaSet;(.*)-[^-]+$
      target_label: source_workload_name
```

#### グラファナ

次のセクションでは、Prometheus インスタンスが実行中の Grafana インスタンスのデータ ソースとして既に構成されていることを前提としています。 Grafana インスタンスを作成および構成する方法の例については、[Prometheus and Grafana](/docs/demos/prometheus_grafana) デモを参照してください。

##### osm-edge ダッシュボードのインポート

osm-edge ダッシュボードは、[our repository](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/charts/osm/grafana/dashboards)から入手できます。これは、Web 管理ポータルで json blob としてインポートできます。

osm-edge ダッシュボードをインポートするための詳細な手順は、[Prometheus and Grafana](/docs/demos/prometheus_grafana) デモにあります。 事前構成されたダッシュボードの概要については、[osm-edge Grafana-dashboards](#osm-grafana-dashboards) を参照してください。
## メトリクスのスクレイピング

メトリクス スクレイピングは、「osm metrics」コマンドを使用して構成できます。 デフォルトでは、osm-edge はメッシュ内のポッドのメトリック スクレイピングを構成しません**。 メトリクスのスクレイピングは、名前空間のスコープで有効または無効にできます。これにより、構成された名前空間に属するポッドがメトリクスのスクレイピングを有効または無効にできます。

メトリクスをスクレイピングするには、次の前提条件を満たす必要があります。

- 名前空間はメッシュの一部である必要があります。 適切なメッシュ名で「openservicemesh.io/monitored-by」ラベルを付ける必要があります。 これは、「osm namespace add」コマンドを使用して実行できます。
- Prometheus エンドポイントをスクレイピングできる実行中のサービス。 osm-edge は [automatic bringup of Prometheus](#automatic-provisioning) の構成を提供します。 または、ユーザーは[bring their own Prometheus](#prometheus)こともできます。
メトリクス スクレイピング用に 1 つ以上の名前空間を有効にするには:

```bash
osm metrics enable --namespace test
osm metrics enable --namespace "test1, test2"

```

メトリクス スクレイピング用に 1 つ以上の名前空間を無効にするには:

```bash
osm metrics disable --namespace test
osm metrics disable --namespace "test1, test2"
```

名前空間でメトリクス スクレイピングを有効にすると、osm-injector によってその名前空間のポッドに次の注釈が追加されます。
```yaml
prometheus.io/scrape: true
prometheus.io/port: 15010
prometheus.io/path: /stats/prometheus
```

## 利用可能な指標

osm-edge は、メッシュ内のトラフィックに関するメトリックと、コントロール プレーンに関するメトリックをエクスポートします。

### カスタム Pipy メトリクス

[SMI Metrics Specifica][7] を実装するために、osm-edge の Pipy プロキシは HTTP トラフィックの次の統計を生成します。

`osm_request_total`: 各プロキシ リクエストで自己インクリメントするカウンタ メトリック。 このメトリクスをクエリすると、メッシュ内のサービスに対するリクエストの成功率と失敗率を確認できます。

`osm_request_duration_ms`: ミリ秒単位でプロキシ リクエストの期間を示すヒストグラム メトリック。 このメトリクスは、メッシュ内のサービス間のレイテンシを理解するためにクエリされます。

どちらの指標にも次のラベルが付いています。

`source_kind`: リクエストを生成したワークロードの Kubernetes リソース タイプ。 Deployment、DaemonSetなど。

`destination_kind`: 要求されたワークロードを処理する Kubernetes リソース タイプ。 Deployment、DaemonSetなど。

`source_name`: 要求されたワークロードを生成した Kubernetes の名前。

`destination_name`: 要求されたワークロードを処理した Kubernetes の名前。

`source_pod`: リクエストを生成した Kubernetes 内のポッドの名前。

`destination_pod`: Kubernetes でリクエストを処理したポッドの名前。

`source_namespace`: リクエストを生成したワークロードの Kubernetes の名前空間。

`destination_namespace`: リクエストを処理したワークロードの Kubernetes の名前空間。

さらに、「osm_request_total」メトリックには、リクエストの HTTP ステータス コードを示す「response_code」タグがあります。 「200」「404」など

### コントロール プレーン

次のメトリクスは、osm-edge コントロール プレーン コンポーネントによって Prometheus 形式で公開されます。 「osm-controller」および「osm-injector」ポッドには、次の Prometheus アノテーションがあります。

```yaml
annotations:
   prometheus.io/scrape: 'true'
   prometheus.io/port: '9091'
```
| メトリック                                | タイプ      | ラベル                                  | 説明                          |
| ------------------------------------- | --------- | --------------------------------------- | -------------------------------------------------------------------------------- |
| osm_k8s_api_event_count               | Count     | type, namespace                         | Number of events received from the Kubernetes API Server                         |
| osm_proxy_connect_count               | Gauge     |                                         | Number of proxies connected to osm-edge controller                                    |
| osm_proxy_reconnect_count             | Count     |                                         | IngressGateway defines the certificate specification for an ingress gateway      |
| osm_proxy_response_send_success_count | Count     | common_name, type                       | Number of responses successfully sent to proxies                                 |
| osm_proxy_response_send_error_count   | Count     | common_name, type                       | Number of responses that errored when being set to proxies                       |
| osm_proxy_config_update_time          | Histogram | resource_type, success                  | Histogram to track time spent for proxy configuration                            |
| osm_proxy_broadcast_event_count       | Count     |                                         | Number of ProxyBroadcast events published by the osm-edge controller                  |
| osm_proxy_xds_request_count           | Count     | common_name, type                       | Number of XDS requests made by proxies                                           |
| osm_proxy_max_connections_rejected    | Count     |                                         | Number of proxy connections rejected due to the configured max connections limit |
| osm_cert_issued_count                 | Count     |                                         | Total number of XDS certificates issued to proxies                               |
| osm_cert_issued_time                  | Histogram |                                         | Histogram to track time spent to issue xds certificate                           |
| osm_admission_webhook_response_total  | Count     | kind, success                           | Total number of admission webhook responses generated                            |
| osm_error_err_code_count              | Count     | err_code                                | Number of errcodes generated by osm-edge                                              |
| osm_http_response_total               | Count     | code, method, path                      | Number of HTTP responses sent                                                    |
| osm_http_response_duration            | Histogram | code, method, path                      | Duration in seconds of HTTP responses sent                                       |
| osm_feature_flag_enabled              | Gauge     | feature_flag                            | Represents whether a feature flag is enabled (1) or disabled (0)                 |
| osm_conversion_webhook_resource_total | Count     | kind, success, from_version, to_version | Number of resources converted by conversion webhooks                             |

#### エラー コード メトリック

osm-edge コントロール プレーンでエラーが発生すると、関連する osm-edge エラー コードに対して ErrCodeCounter Prometheus メトリックが増加します。 エラー コードとその説明の完全なリストについては、[osm-edge Control Plane Error Code Troubleshooting Guide](/docs/guides/troubleshooting/control_plane_error_codes) を参照してください。

エラー コード メトリックの完全修飾名は「osm_error_err_code_count」です。
> 注: プロセスの再起動につながるエラーに対応するメトリックは、時間内にスクレイピングされない場合があります。

## Prometheus からメトリックをクエリする

### あなたが始める前に

[osm-edge Demo][2] を実行する手順に従っていることを確認してください

### リクエスト数のプロキシ メトリックのクエリ

1. Prometheus サービスがクラスターで実行されていることを確認する
    - kubernetes で、次のコマンドを実行します: `kubectl get svc osm-prometheus -n <osm-namespace>`。
     ![image](https://user-images.githubusercontent.com/59101963/85906800-478b3580-b7c4-11ea-8eb2-63bd83647e5f.png)
   - 注: `<osm-namespace>` は、osm コントロール プレーンがインストールされている名前空間を指します。
2. プロメテウス UI を開く
    - リポジトリのルートにいることを確認し、次のスクリプトを実行します: `./scripts/port-forward-prometheus.sh`
    - Web ブラウザーで次の URL [http://localhost:7070][5] にアクセスします。
3.Prometheus クエリを実行する
    - Web ページの上部にある [式] 入力ボックスに、テキスト `sidecar_cluster_upstream_rq_xx{sidecar_response_code_class="2"}` を入力し、実行ボタンをクリックします。
    - このクエリは、成功した HTTP リクエストを返します

サンプル結果は次のようになります。
![image](https://user-images.githubusercontent.com/59101963/85906690-f24f2400-b7c3-11ea-89b2-a3c42041c7a0.png)

## Grafana でメトリクスを視覚化する

### Grafana ダッシュボードを表示するための前提条件のリスト

[osm-edge Demo][2] を実行する手順に従っていることを確認してください

### サービス間メトリックの Grafana ダッシュボードの表示

1. Prometheus サービスがクラスターで実行されていることを確認する
    - kubernetes で、次のコマンドを実行します: `kubectl get svc osm-prometheus -n <osm-namespace>`
     ![image](https://user-images.githubusercontent.com/59101963/85906800-478b3580-b7c4-11ea-8eb2-63bd83647e5f.png)
2. Grafana サービスがクラスターで実行されていることを確認する
    - kubernetes で、次のコマンドを実行します: `kubectl get svc osm-grafana -n <osm-namespace>`
     ![image](https://user-images.githubusercontent.com/59101963/85906847-70abc600-b7c4-11ea-853d-f4c9b188ab9f.png)
3. Grafana UI を開く
    - リポジトリのルートにいることを確認し、次のスクリプトを実行します: `./scripts/port-forward-grafana.sh`
    - Web ブラウザーで次の URL [http://localhost:3000][4] にアクセスします。
4. Grafana UI はログインの詳細を要求します。次のデフォルト設定を使用します。
    - ユーザー名: 管理者
    - パスワード: 管理者
5. サービス間メトリックの Grafana ダッシュボードの表示
    - Grafana のダッシュボードの左隅のナビゲーション メニューから、フォルダー osm-edge Data Plane の osm-edge Service to Service ダッシュボードに移動できます。
    - または、Web ブラウザーで次の URL [http://localhost:3000/d/OSMs2sMetrics/osm-service-to-service-metrics?orgId=1][6] にアクセスします。

osm-edge Service to Service Metrics ダッシュボードは次のようになります。
![image](https://user-images.githubusercontent.com/59101963/85907233-a604e380-b7c5-11ea-95b5-9190fbc7967f.png)

## osm-edge Grafana ダッシュボード

osm-edge は、Prometheus によってキャプチャされたサービス関連の情報を表示および追跡するために、いくつかの事前に調理された Grafana ダッシュボードを提供します。

1.osm-edge データ プレーン
    - **osm-edge Data Plane Performance Metrics**: このダッシュボードでは、osm-edge のデータ プレーンのパフォーマンスを表示できます
     ![image](https://user-images.githubusercontent.com/64559656/138173256-28011b16-cace-4365-b166-db909543472e.png)
   - **osm-edge Service to Service Metrics**: このダッシュボードでは、特定のソース サービスから特定の宛先サービスへのトラフィック メトリックを表示できます。
     ![image](https://user-images.githubusercontent.com/64559656/141853912-10ec3767-3d5b-40e8-8f13-d39a32980183.png)
   - **osm-edge Pod to Service Metrics**: このダッシュボードでは、Pod から接続/通信するすべてのサービスへのトラフィック メトリックを調査できます。
     ![image](https://user-images.githubusercontent.com/64559656/140724337-0568dde0-e6c5-4764-8b6f-c1fcaf144b4e.png)
   - **osm-edge Workload to Service Metrics**: このダッシュボードは、ワークロード (デプロイメント、replicaSet) から接続/通信するすべてのサービスへのトラフィック メトリックを提供します。
     ![image](https://user-images.githubusercontent.com/64559656/140724800-8152cb8b-1617-4866-b008-f12c31f702c2.png)
   - **osm-edge Workload to Workload Metrics**: このダッシュボードには、ワークロードからワークロードへのメッシュ内のリクエストのレイテンシが表示されます
     ![image](https://user-images.githubusercontent.com/64559656/140718968-b3999e30-e6d1-4d95-b07b-0043595aca71.png)

2. osm-edge コントロール プレーン
   - **osm-edge コントロール プレーン メトリック**: このダッシュボードは、特定のサービスから osm-edge のコントロール プレーンへのトラフィック メトリックを提供します。
     ![image](https://user-images.githubusercontent.com/64559656/138173115-0a012450-0d91-449d-9c09-975b68fde03d.png)
   - **Mesh and Pipy Details**: このダッシュボードでは、osm-edge のコントロール プレーンのパフォーマンスと動作を表示できます
     ![image](https://user-images.githubusercontent.com/64559656/141852750-61da99ac-a431-4251-bd97-8aa4601232c3.png)

[1]: https://prometheus.io/docs/introduction/overview/
[2]: https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/README.md
[3]: https://grafana.com/docs/grafana/latest/getting-started/#what-is-grafana
[4]: http://localhost:3000
[5]: http://localhost:7070
[6]: http://localhost:3000/d/OSMs2sMetrics/osm-service-to-service-metrics?orgId=1
[7]: https://github.com/servicemeshinterface/smi-spec/blob/master/apis/traffic-metrics/v1alpha1/traffic-metrics.md
[8]: https://grafana.com/oss/grafana/
