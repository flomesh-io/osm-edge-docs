---
title: "osm-edge を Prometheus および Grafana と統合する"
description: "独自の Prometheus および Grafana スタックを使用して、osm-edge 固有の設定とダッシュボードをセットアップする方法について説明する"
type: ドキュメント
weight: 25
---

# osm-edge を独自の Prometheus および Grafana スタックと統合する

次の記事では、サンプルの独自の (BYO) Prometheus および Grafana スタックをクラスターに作成し、osm-edge の可観測性と監視のためにそのスタックを設定する方法を示す。osm-edge を使用した Prometheus および Grafana スタックの自動プロビジョニングの使用例については、[Observability](https://docs.openservicemesh.io/docs/getting_started/observability/)入門ガイドを参照してください。

> 重要: この記事で作成した設定は、本番環境では使用しないでください。プロダクショングレードのデプロイについては、[Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md)と [Deploy Grafana in Kubernetes](https://grafana.com/docs/grafana/latest/installation/kubernetes/)を参照してください。

## 前提条件
- Kubernetes {{< param min_k8s_version >}}あるいはそれより高いバージョンを実行しているKubernetesクラスター。
- Kubernetes クラスターにosm-edgeがインストールされている。
- `kubectl` がインストールされ、クラスターの API サーバーにアクセスできる。
- `osm` CLIがインストールされている。
- `helm` CLIがインストールされている。

## サンプルの Prometheus インスタンスをデプロイする

`helm` を使用して、Prometheus インスタンスをデフォルトの名前空間のクラスターにデプロイする。

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install stable prometheus-community/prometheus
```

`helm install` コマンドの出力には、Prometheus サーバーの DNS 名が含まれている。例えば: 

```console
...
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
stable-prometheus-server.metrics.svc.cluster.local
...
```

後の手順で使用するために、この DNS 名を記録する。

## osm-edge 用に Prometheus を設定する

Prometheusは、osm-edgeのエンドポイントをスケープし、osm-edgeのラベリング、リラベリング、エンドポイント設定を適切に処理できるよう設定する必要がある。この設定は、後のステップで設定するosm-edge Grafanaダッシュボードが、osm-edgeからかき集めたデータを適切に表示するためにも役立つ。

kubectl get configmap` を使用して、`stable-prometheus-sever` のコンフィグマップが作成されたことを確認する。例えば

```bash
$ kubectl get configmap

NAME                             DATA   AGE
...
stable-prometheus-alertmanager   1      18m
stable-prometheus-server         5      18m
...
```

以下を使用して「update-prometheus-configmap.yaml」を作成する。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: stable-prometheus-server
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
      evaluation_interval: 30s

    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          # TODO need to remove this when the CA and SAN match
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        metric_relabel_configs:
        - source_labels: [__name__]
          regex: '(apiserver_watch_events_total|apiserver_admission_webhook_rejection_count)'
          action: keep
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        metric_relabel_configs:
        - source_labels: [__name__]
          regex: '(sidecar_server_live|sidecar_cluster_health_check_.*|sidecar_cluster_upstream_rq_xx|sidecar_cluster_upstream_cx_active|sidecar_cluster_upstream_cx_tx_bytes_total|sidecar_cluster_upstream_cx_rx_bytes_total|sidecar_cluster_upstream_rq_total|sidecar_cluster_upstream_cx_destroy_remote_with_active_rq|sidecar_cluster_upstream_cx_connect_timeout|sidecar_cluster_upstream_cx_destroy_local_with_active_rq|sidecar_cluster_upstream_rq_pending_failure_eject|sidecar_cluster_upstream_rq_pending_overflow|sidecar_cluster_upstream_rq_timeout|sidecar_cluster_upstream_rq_rx_reset|^osm.*)'
          action: keep
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
        # for non-ReplicaSets (DaemonSet, StatefulSet)
        # __meta_kubernetes_pod_controller_kind=DaemonSet
        # __meta_kubernetes_pod_controller_name=foo
        # =>
        # workload_kind=DaemonSet
        # workload_name=foo
        - source_labels: [__meta_kubernetes_pod_controller_kind]
          action: replace
          target_label: source_workload_kind
        - source_labels: [__meta_kubernetes_pod_controller_name]
          action: replace
          target_label: source_workload_name
        # for ReplicaSets
        # __meta_kubernetes_pod_controller_kind=ReplicaSet
        # __meta_kubernetes_pod_controller_name=foo-bar-123
        # =>
        # workload_kind=Deployment
        # workload_name=foo-bar
        # deplyment=foo
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

      - job_name: 'smi-metrics'
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
        metric_relabel_configs:
        - source_labels: [__name__]
          regex: 'sidecar_.*osm_request_(total|duration_ms_(bucket|count|sum))'
          action: keep
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_(\d{3})_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: response_code
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_\d{3}_source_namespace_(.*)_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: source_namespace
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_\d{3}_source_namespace_.*_source_kind_(.*)_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: source_kind
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_(.*)_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: source_name
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_(.*)_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: source_pod
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_(.*)_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: destination_namespace
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_(.*)_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: destination_kind
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_(.*)_destination_pod_.*_osm_request_total
          target_label: destination_name
        - source_labels: [__name__]
          action: replace
          regex: sidecar_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_(.*)_osm_request_total
          target_label: destination_pod
        - source_labels: [__name__]
          action: replace
          regex: .*(osm_request_total)
          target_label: __name__

        - source_labels: [__name__]
          action: replace
          regex: sidecar_source_namespace_(.*)_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: source_namespace
        - source_labels: [__name__]
          action: replace
          regex: sidecar_source_namespace_.*_source_kind_(.*)_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: source_kind
        - source_labels: [__name__]
          action: replace
          regex: sidecar_source_namespace_.*_source_kind_.*_source_name_(.*)_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: source_name
        - source_labels: [__name__]
          action: replace
          regex: sidecar_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_(.*)_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: source_pod
        - source_labels: [__name__]
          action: replace
          regex: sidecar_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_(.*)_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: destination_namespace
        - source_labels: [__name__]
          action: replace
          regex: sidecar_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_(.*)_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: destination_kind
        - source_labels: [__name__]
          action: replace
          regex: sidecar_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_(.*)_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: destination_name
        - source_labels: [__name__]
          action: replace
          regex: sidecar_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_(.*)_osm_request_duration_ms_(bucket|sum|count)
          target_label: destination_pod
        - source_labels: [__name__]
          action: replace
          regex: .*(osm_request_duration_ms_(bucket|sum|count))
          target_label: __name__

      - job_name: 'kubernetes-cadvisor'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
        - role: node
        metric_relabel_configs:
        - source_labels: [__name__]
          regex: '(container_cpu_usage_seconds_total|container_memory_rss)'
          action: keep
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

「kubectl apply」を使用して、Prometheus サーバーの configmap を更新する。

```bash
kubectl apply -f update-prometheus-configmap.yaml
```

「kubectl port-forward」を使用して Prometheus 管理アプリケーションと開発用コンピューターの間でトラフィックを転送することにより、Prometheus が osm-edgeメッシュと API エンドポイントをスクレイプできることを確認する。

```bash
export POD_NAME=$(kubectl get pods -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 9090
```

ウェブブラウザで http://localhost:9090/targets にアクセスし、Prometheus 管理アプリケーションにアクセスして、エンドポイントが接続され、起動しており、スクラップが実行されていることを確認する。

<p align="center">
  <img src="/docs/images/byo_guide/prom_targets.png" width="100%"/>
</p>
<center><i>osm-edgeで設定された特定のリラベル設定を持つターゲットは "up "になるはずだ。</i></center><br>

ポートフォワーディングコマンドを停止する。

## Grafanaインスタンスをデプロイする

Helm` を使用して、Grafana インスタンスをデフォルトの名前空間でクラスタにデプロイする。

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana/grafana --generate-name
```

Grafana の管理者パスワードを表示するには、「kubectl get secret」を使用する。

```bash
export SECRET_NAME=$(kubectl get secret -l "app.kubernetes.io/name=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl get secret $SECRET_NAME -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

「kubectl port-forward」を使用して、Grafana の管理アプリケーションと開発用コンピューターの間でトラフィックを転送する。

```bash
export POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 3000
```

ウェブブラウザーを開いて `http://localhost:3000` にアクセスし、Grafana の管理アプリケーションにアクセスする。前の手順のユーザー名と管理者パスワードとして admin を使用する。エンドポイントの接続、稼働、廃棄が実行されていることを確認する。

管理アプリケーションから:

*「Settings」、「Data Sources」の順で選択する。
*「Add data source」を選択する。
*「Prometheus」データソースを見つけて、「Select」を選択する。
* 前のステップの URL に、stable-prometheus-server.default.svc.cluster.local などの DNS 名を入力する。

[Save and Test] を選択し、[Data source is working] と表示されていることを確認する。

## osm-edgeダッシュボードをインポートする

osm-edge ダッシュボードは[osm-edge GitHub repository](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/charts/osm/grafana/dashboards) から利用でき、管理アプリケーションでjson blobとしてインポートすることが可能だ。

ダッシュボードをインポートするには:
* 「+」の上にカーソルを置き、「Import」を選択する。
*[osm-mesh-sidecar-details dashboard](https://raw.githubusercontent.com/flomesh-io/osm-edge/{{< param osm_branch >}}/charts/osm/grafana/pipy/dashboards/osm-mesh-sidecar-details.json) から JSON をコピーし、「Import via panel json」に貼り付ける。
*「Load」を選択する。
*「Import」を選択する。

「Mesh and Sidecar Details」ダッシュボードが作成されていることを確認する。
