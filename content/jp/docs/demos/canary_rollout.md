---
title: "SMIトラフィック分割を使用したカナリアロールアウト"
description: "SMIトラフィック分割を使用したカナリアロールアウトの管理"
type: docs
weight: 21
---

このガイドでは、SMI トラフィック分割構成を使用してカナリア ロールアウトを実行する方法を示す。

## 前提条件

- Kubernetes{{< param min_k8s_version >}}あるいはそれ以上を実行している Kubernetesクラスター。
- osm-edgeがインストールされている。
- APIサーバーとやり取りするためのkubectlは使用可能。
- サービスメッシュを管理するための「osm」CLIは使用可能。


##　デモ

このデモでは、HTTPアプリケーションを展開し、カナリアロールアウトを実行する。このロールアウトでは、新しいバージョンのアプリケーションが展開され、サービスに向けられたトラフィックの一部を処理する。

トラフィックを複数のサービス バックエンドに分割するには、 [SMI Traffic Split API] (https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md) が使用される。この API の使用方法の詳細については、[traffic split guide](docs/guides/traffic_management/traffic_split)を参照してください。クライアントアプリケーションが透過的にトラフィックを複数のサービスバックエンドに分割するには、クライアントアプリケーションが TrafficSplitリソースで参照されるルートサービスの FQDN にトラフィックを転送する必要があることに注意することが重要だ。このデモでは、「curl」クライアントはトラフィックを「httpbin」ルートサービスに転送します。最初はサービスのバージョン「v1」によってバックアップされ、その後、カナリアロールアウトを実行してトラフィックの一部をサービスのバージョン「v2」に転送する。

次の手順は、カナリアロールアウトのデプロイメント戦略を示している。
> 注意:[Permissive traffic policy mode](docs/guides/traffic_management/permissive_mode)を有効にして、明示的なアクセス制御ポリシーを作成する必要がないようにする。

1. permissive modeを有効にする

    ```bash
    osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

2. その名前空間をメッシュに登録した後、`curl` クライアントを `curl` 名前空間にデプロイする。

    ```bash
     # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    `curl` クライアントポッドが稼働中であることを確認する。
    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

３. クライアントがトラフィックを転送するルート `httpbin` サービスを作成する。このサービスにはセレクター「app: httpbin」がある。

    ``` # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

    # Create the httpbin root service and service account
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/canary/httpbin.yaml -n httpbin
    ```

1.  `httpbin`サービスのバージョン`v1` をデプロイする。サービス httpbin-v1 にはセレクター アプリ: httpbin, バージョン: v1 があり、デプロイメント httpbin-v1 にはラベル アプリ: httpbin, バージョン: v1 があり、httpbin ルートサービスと httpbin-v1 サービスの両方のセレクターと一致する。

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/canary/httpbin-v1.yaml -n httpbin
    ```

２. すべてのトラフィックを「httpbin-v1」サービスに転送する SMI トラフィック分割リソースを作成する。
    ```kubectl apply -f - <<EOF
    apiVersion: split.smi-spec.io/v1alpha2
    kind: TrafficSplit
    metadata:
      name: http-split
      namespace: httpbin
    spec:
      service: httpbin.httpbin.svc.cluster.local
      backends:
      - service: httpbin-v1
        weight: 100
    EOF
    ```

1. ルートサービス FQDN httpbin.httpbin.svc.cluster.local に向けられたすべてのトラフィックが httpbin-v1 ポッドにルーティングされていることを確認する。これは、HTTP 応答ヘッダーを調べて、その要求が成功し、そして表示されるポッドが httpbin-v1 に対応していることを確認することで検証できる。

    ```console
    for i in {1..10}; do kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -sI http://httpbin.httpbin:14001/json | egrep 'HTTP|pod'; done
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    ```

    上記の出力は、10 個のリクエストすべてが HTTP 200 OK を返し、`httpbin-v1` ポッドによって応答されたことを示している。

1. 「httpbin」サービスのバージョン「v2」をデプロイして、カナリアロールアウトを準備する。サービス `httpbin-v2` にはセレクタ `アプリ: httpbin, バージョン: v2` があり、デプロイメント `httpbin-v2` にはラベル `アプリ: httpbin, バージョン: v2` があり、両方の `httpbin` ルートサービスと「httpbin-v2」サービスのセレクタに一致する。

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/canary/httpbin-v2.yaml -n httpbin
    ```

1. SMIトラフィック分割リソースを更新して、ルートサービスの FQDN httpbin.httpbin.svc.cluster.local に向けられたトラフィックを httpbin-v1 と httpbin-v2 の両方のサービスに分割し、v1 と v2 のバージョンの httpbin サービスの前にそれぞれカナリアロールアウトを配置し実行する。 ウエイトを均等に分散して、トラフィックの分割を示す。

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: split.smi-spec.io/v1alpha2
    kind: TrafficSplit
    metadata:
      name: http-split
      namespace: httpbin
    spec:
      service: httpbin.httpbin.svc.cluster.local
      backends:
      - service: httpbin-v1
        weight: 50
      - service: httpbin-v2
        weight: 50
    EOF
    ```

1. バックエンドサービスに割り当てられたウエイトに比例してトラフィックが分割されることを確認する。「v1」と「v2」の両方に「50」のウエイトを設定したため、以下に示すように、リクエストは両方のバージョンに負荷分散する必要がある。

    ```console
    $ for i in {1..10}; do kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -sI http://httpbin.httpbin:14001/json | egrep 'HTTP|pod'; done
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    ```

    
    上記の出力は、10 個のリクエストすべてが HTTP 200 OK を返し、「httpbin-v1」ポッドと「httpbin-v2」ポッドの両方がそれぞれ「TrafficSplit」構成で割り当てられたウエイトに基づいて 5つのリクエストに応答したことを示している。