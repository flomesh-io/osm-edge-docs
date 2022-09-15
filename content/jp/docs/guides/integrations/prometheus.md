---
title: 「Prometheus を osm-edge と統合する」
description: 「osm-edge がメトリクスのために Prometheus とどのように統合されるかを示す簡単なデモ」
aliases: "/docs/integrations/demo_prometheus"
type: docs
weight: 3
---

# Prometheus を osm-edge と統合する

## Prometheus と osm-edge の統合

osm-edge が Prometheus でどのように機能するかを理解するには、サンプル アプリケーションを含む新しいメッシュをインストールして、どのメトリックが収集されるかを確認してください。

1. 独自の Prometheus インスタンスを使用して osm-edge をインストールします。

   ```console
   $ osm install --set osm.deployPrometheus=true,osm.enablePermissiveTrafficPolicy=true
   osm-edge installed successfully in namespace [osm-system] with mesh name [osm]
   ```

1. サンプル ワークロード用の名前空間を作成します。

   ```console
   $ kubectl create namespace metrics-demo
   namespace/metrics-demo created
   ```

1. 新しい osm-edge モニターを新しい名前空間にします:

   ```console
   $ osm namespace add metrics-demo
   Namespace [metrics-demo] successfully added to mesh [osm]
   ```

1. osm-edge の Prometheus を構成して、新しい名前空間からメトリクスをスクレイピングします。

   ```console
   $ osm metrics enable --namespace metrics-demo
   Metrics successfully enabled in namespace [metrics-demo]
   ```

1. サンプル アプリケーションをインストールします。

   ```console
   $ kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n metrics-demo
   serviceaccount/curl created
   deployment.apps/curl created
   $ kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n metrics-demo
   serviceaccount/httpbin created
   service/httpbin created
   deployment.apps/httpbin created
   ```

   新しい Pod が実行中で、すべてのコンテナーの準備が整っていることを確認します。

   ```console
   $ kubectl get pods -n metrics-demo
   NAME                       READY   STATUS    RESTARTS   AGE
   curl-54ccc6954c-q8s89      2/2     Running   0          95s
   httpbin-8484bfdd46-vq98x   2/2     Running   0          72s
   ```

1. トラフィックを生成します。

    次のコマンドは、curl Pod が httpbin Pod に対して 1 秒あたり約 1 回のリクエストを永遠に行うようにします。

   ```console
   $ kubectl exec -n metrics-demo -ti "$(kubectl get pod -n metrics-demo -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'while :; do curl -i httpbin.metrics-demo:14001/status/200; sleep 1; done'
   HTTP/1.1 200 OK
   server: gunicorn/19.9.0
   date: Wed, 06 Jul 2022 02:53:16 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   connection: keep-alive

   HTTP/1.1 200 OK
   server: gunicorn/19.9.0
   date: Wed, 06 Jul 2022 02:53:17 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   connection: keep-alive
   ...
   ```

1. Prometheus でメトリックを表示します。

    Prometheus ポートを転送します。

   ```console
   $ kubectl port-forward -n osm-system $(kubectl get pods -n osm-system -l app=osm-prometheus -o jsonpath='{.items[0].metadata.name}') 7070
   Forwarding from 127.0.0.1:7070 -> 7070
   Forwarding from [::1]:7070 -> 7070
   ```

   Web ブラウザーで http://localhost:7070 に移動して、Prometheus UI を表示します。 次のクエリは、curl ポッドから httpbin ポッドに対して行われる 1 秒あたりのリクエスト数を示しています。これは約 1 である必要があります。

   ```
   irate(sidecar_cluster_upstream_rq_xx{source_service="curl", sidecar_cluster_name="metrics-demo/httpbin"}[30s])
   ```

   Prometheus UI 内から利用できる他のメトリクスを自由に調べてください。

1. 掃除

    デモ リソースの使用が完了したら、最初にアプリケーションの名前空間を削除してクリーンアップします。

   ```console
   $ kubectl delete ns metrics-demo
   namespace "metrics-demo" deleted
   ```

   次に、osm-edge をアンインストールします。

   ```
   $ osm uninstall mesh
   Uninstall osm-edge [mesh name: osm] ? [y/n]: y
   osm-edge [mesh name: osm] uninstalled
   ```

   アンインストール後に osm-edge のクラスター全体のリソースを削除するには、次のコマンドを実行します。 詳細なコンテキストと情報については、[uninstall guide](/docs/guides/uninstall/) を参照してください。
   ```console
   $ osm uninstall mesh --delete-cluster-wide-resources
   ```