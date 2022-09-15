---
title: "メッシュ外部の宛先の回路遮断"
description: "メッシュ外部の宛先の回路遮断の設定"
type: ドキュメント
weight: 22
---

このガイドでは、osm-edge マネージド サービスメッシュの外部にある宛先の回路遮断を設定する方法を示す。

## 前提条件
- Kubernetes{{< param min_k8s_version >}}あるいはそれ以上を実行している Kubernetesクラスター。
- osm-edgeがインストールされている。
- APIサーバーとやり取りするためのkubectlは使用可能。
- サービスメッシュを管理するための「osm」CLIは使用可能。
- osm-edge バージョン >= v1.1.0.

## デモ
次のデモは、サービスメッシュの外部にある httpbin サービスにトラフィックを送信する負荷テストクライアント [fortio](https://github.com/fortio/fortio)を示している。メッシュ外部のトラフィックは[Egress](/docs/guides/traffic_management/egress)トラフィックとして扱われ、 [Egress traffic policy](/docs/guides/traffic_management/egress/#1-configuring-egress-policies)を使用して承認される。外部 httpbin サービスへのトラフィックに回路遮断を適用すると、設定された回路遮断がトリップしたときに fortio クライアントにどのような影響があるかを確認する。

1.  `httpbin` サービスを `httpbin` 名前空間にデプロイする。「httpbin」サービスはポート「14001」で実行され、メッシュに追加されないため、メッシュの外部の宛先と見なされる。

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Deploy httpbin service in the httpbin namespace
    kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    「httpbin」サービスとポッドが稼働中であることを確認する。

    ```console
    $ kubectl get svc -n httpbin
    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    httpbin   ClusterIP   10.96.198.23   <none>        14001/TCP   20s
    ```

    ```console
    $ kubectl get pods -n httpbin
    NAME                     READY   STATUS    RESTARTS   AGE
    httpbin-5b8b94b9-lt2vs   1/1     Running   0          20s
    ```

1. その名前空間をメッシュに登録した後、「fortio」負荷テスト クライアントを「client」名前空間にデプロイする。
    ```bash
    # Create the client namespace
    kubectl create namespace client

    # Add the namespace to the mesh
    osm namespace add client

    # Deploy fortio client in the client namespace
    kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n client
    ```

    クライアントポッドが稼働中であることを確認する。

    ```console
    $ kubectl get pods -n client
    NAME                      READY   STATUS    RESTARTS   AGE
    fortio-6477f8495f-bj4s9   2/2     Running   0          19s
    ```

1. クライアント名前空間の fortio クライアントが外部の httpbin サービスと通信できるようにするエグレスポリシーを構成する。HTTP要求はポート14001のホスト httpbin.httpbin.svc.cluster.local に送信される。
   
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-external
      namespace: client
    spec:
      sources:
      - kind: ServiceAccount
        name: default
        namespace: client
      hosts:
      - httpbin.httpbin.svc.cluster.local
      ports:
      - number: 14001
        protocol: http
    EOF
    ```

2. fortioクライアントがポート14001 で外部ホスト httpbin.httpbin.svc.cluster.local サービスに HTTP 要求を正常に作成できることを確認する。
 `5` の同時接続 (`-c 5`) で外部サービスを呼び出し、`50` のリクエスト (`-n 50`) を送信する。
   
    ```console
    $ export fortio_pod="$(kubectl get pod -n client -l app=fortio -o jsonpath='{.items[0].metadata.name}')"

    $ kubectl exec "$fortio_pod" -c fortio -n client -- /usr/bin/fortio load -c 5 -qps 0 -n 50 -loglevel Warning http://httpbin.httpbin.svc.cluster.local:14001/get
    19:56:34 I logger.go:127> Log level is now 3 Warning (was 2 Info)
    Fortio 1.17.1 running at 0 queries per second, 8->8 procs, for 50 calls: http://httpbin.httpbin.svc.cluster.local:14001/get
    Starting at max qps with 5 thread(s) [gomax 8] for exactly 50 calls (10 per thread + 0)
    Ended after 36.3659ms : 50 calls. qps=1374.9
    Aggregated Function Time : count 50 avg 0.003374618 +/- 0.0007546 min 0.0013124 max 0.0066215 sum 0.1687309
    # range, mid point, percentile, count
    >= 0.0013124 <= 0.002 , 0.0016562 , 4.00, 2
    > 0.002 <= 0.003 , 0.0025 , 10.00, 3
    > 0.003 <= 0.004 , 0.0035 , 86.00, 38
    > 0.004 <= 0.005 , 0.0045 , 98.00, 6
    > 0.006 <= 0.0066215 , 0.00631075 , 100.00, 1
    # target 50% 0.00352632
    # target 75% 0.00385526
    # target 90% 0.00433333
    # target 99% 0.00631075
    # target 99.9% 0.00659043
    Sockets used: 5 (for perfect keepalive, would be 5)
    Jitter: false
    Code 200 : 50 (100.0 %)
    Response Header Sizes : count 50 avg 230 +/- 0 min 230 max 230 sum 11500
    Response Body/Total Sizes : count 50 avg 460 +/- 0 min 460 max 460 sum 23000
    All done 50 calls (plus 0 warmup) 3.375 ms avg, 1374.9 qps
    ```

    上記のように、すべてのリクエストが成功した。
    ```
    Code 200 : 50 (100.0 %)
    ```

3. 次に、外部ホスト httpbin.httpbin.svc.cluster.local に送信されるトラフィックの UpstreamTrafficSetting リソースを使用して回路遮断の構成を適用し、同時接続とリクエストの最大数を 1 に制限する。外部 (エグレス) トラフィックに「UpstreamTrafficSetting」構成を適用する場合、「UpstreamTrafficSetting」リソースも「Egress」構成でマッチとして指定され、マッチング「Egress」リソースと同じ名前空間に属している必要がある。これは、外部トラフィックに対して回路遮断の制限を適用するために必要だ。したがって、以前に適用された「Egress」構成も更新して、「matches」フィールドを指定する。

    
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: policy.openservicemesh.io/v1alpha1
    kind: UpstreamTrafficSetting
    metadata:
      name: httpbin-external
      namespace: client
    spec:
      host: httpbin.httpbin.svc.cluster.local
      connectionSettings:
        tcp:
          maxConnections: 1
        http:
          maxPendingRequests: 1
          maxRequestsPerConnection: 1
    ---
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-external
      namespace: client
    spec:
      sources:
      - kind: ServiceAccount
        name: default
        namespace: client
      hosts:
      - httpbin.httpbin.svc.cluster.local
      ports:
      - number: 14001
        protocol: http
      matches:
      - apiGroup: policy.openservicemesh.io/v1alpha1
        kind: UpstreamTrafficSetting
        name: httpbin-external
    EOF
    ```

4. 上記で構成された接続およびリクエストレベルの回路遮断の制限により、「fortio」クライアントが以前と同じ量の成功したリクエストを作成できないことを確認する。
    
    ```console
    $ kubectl exec "$fortio_pod" -c fortio -n client -- /usr/bin/fortio load -c 5 -qps 0 -n 50 -loglevel Warning http://httpbin.httpbin.svc.cluster.local:14001/get
    19:58:48 I logger.go:127> Log level is now 3 Warning (was 2 Info)
    Fortio 1.17.1 running at 0 queries per second, 8->8 procs, for 50 calls: http://httpbin.httpbin.svc.cluster.local:14001/get
    Starting at max qps with 5 thread(s) [gomax 8] for exactly 50 calls (10 per thread + 0)
    19:58:48 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [4] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [4] Non ok http code 503 (HTTP/1.1 503)
    Ended after 33.1549ms : 50 calls. qps=1508.1
    Aggregated Function Time : count 50 avg 0.002467842 +/- 0.001827 min 0.0003724 max 0.0067697 sum 0.1233921
    # range, mid point, percentile, count
    >= 0.0003724 <= 0.001 , 0.0006862 , 34.00, 17
    > 0.001 <= 0.002 , 0.0015 , 50.00, 8
    > 0.002 <= 0.003 , 0.0025 , 60.00, 5
    > 0.003 <= 0.004 , 0.0035 , 84.00, 12
    > 0.004 <= 0.005 , 0.0045 , 88.00, 2
    > 0.005 <= 0.006 , 0.0055 , 92.00, 2
    > 0.006 <= 0.0067697 , 0.00638485 , 100.00, 4
    # target 50% 0.002
    # target 75% 0.003625
    # target 90% 0.0055
    # target 99% 0.00667349
    # target 99.9% 0.00676008
    Sockets used: 25 (for perfect keepalive, would be 5)
    Jitter: false
    Code 200 : 29 (58.0 %)
    Code 503 : 21 (42.0 %)
    Response Header Sizes : count 50 avg 133.4 +/- 113.5 min 0 max 230 sum 6670
    Response Body/Total Sizes : count 50 avg 368.02 +/- 108.1 min 241 max 460 sum 18401
    All done 50 calls (plus 0 warmup) 2.468 ms avg, 1508.1 qps
    ```

    上記のように、リクエストの 58% のみが成功し、残りは回路遮断が作動したときに失敗した。
    ```
    Code 200 : 29 (58.0 %)
    Code 503 : 21 (42.0 %)
    ```

5. 「Envoy」サイドカーの統計を調べて、回路遮断を作動させたリクエストに関する統計を確認する。
    
    ```console
    $ osm proxy get stats $fortio_pod -n client | grep 'httpbin.*pending'
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.default.remaining_pending: 1
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.default.rq_pending_open: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.high.rq_pending_open: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_active: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_failure_eject: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_overflow: 21
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_total: 29
    ```

    `cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_overflow: 21 は、21 の要求が回路遮断を作動させたことを示す。これは、前の手順で確認された失敗したリクエストの数と一致している: コード 200: 29 (58.0 %)。