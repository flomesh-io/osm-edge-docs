---
title: "TCPトラフィックルーティング"
description: "TCPトラフィックルーティングをセットアップする"
type: docs
weight: 20
---

このガイドでは、osm-edge のTCPルーティング機能を使用して通信するサービスメッシュ内のTCPクライアントおよびサーバー アプリケーションについて説明する。

## 前提条件
- Kubernetes{{< param min_k8s_version >}}あるいはそれより高いバージョンを実行しているKubernetesクラスター。
- osm-edgeはインストールされている。
-  API サーバーとやり取りするための`kubectl`は使用可能。
- サービスメッシュを管理するための`osm` CLIは使用可能。

## デモ
次のデモは、TCPクライアントが「tcp-echo」サーバーにデータを送信し、TCP接続を介してデータをクライアントにエコーバックする様子を示す。

1. osm-edge がインストールされている名前空間を設定する。
    ```bash
    osm_namespace=osm-system  # Replace osm-system with the namespace where osm-edge is installed if different
    ```

1. `tcp-demo` 名前空間に `tcp-echo` サービスをデプロイする。tcp-echo サービスは、appProtocol フィールドが tcp に設定されたポート9000で実行される。これは、ポート9000 の tcp-echoサービスに向けられたトラフィックに TCPルーティングを使用する必要があることを osm-edge に示す。
    ```bash
    # Create the tcp-demo namespace
    kubectl create namespace tcp-demo

    # Add the namespace to the mesh
    osm namespace add tcp-demo

    # Deploy the service
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/tcp-echo.yaml -n tcp-demo
    ```

    tcp-echo` サービスとポッドが起動していることを確認する。

    ```console
    $ kubectl get svc,po -n tcp-demo
    NAME               TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
    service/tcp-echo   ClusterIP   10.0.216.68   <none>        9000/TCP   97s

    NAME                            READY   STATUS    RESTARTS   AGE
    pod/tcp-echo-6656b7c4f8-zt92q   2/2     Running   0          97s
    ```

1. `curl` クライアントを `curl` 名前空間にデプロイする。

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    curl` クライアントポッドが起動していることを確認する。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

### 許容なトラフィックポリシーモードの使用

[permissive traffic policy mode](docs/guides/traffic_management/permissive_mode)を使用してサービスディスカバリを有効にする。これにより、明示的な SMI ポリシーを必要とせずにアプリケーション接続を確立できる。

1. 許容なトラフィックポリシーモードを有効にする
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}' --type=merge
    ```

1. `curl` クライアントが TCP ルーティングを使用して `tcp-echo` サービスからの応答を送受信できることを確認する。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    echo response: hello
    ```

    「tcp-echo」サービスは、クライアントから送信されたデータをエコーバックする必要がある。上記の例では、クライアントが「hello」を送信し、「tcp-echo」サービスが「echo response: hello」で応答する。

### SMIトラフィックポリシーモードの使用

SMIトラフィックポリシーモードを使用する場合、明示的なトラフィックポリシーを設定して、アプリケーション接続を許可する必要がある。`curl` クライアントがポート `9000` で `tcp-echo` サービスと通信できるように SMI ポリシーを設定する。

1. 許容なトラフィックポリシーモードを無効にして、SMIトラフィックポリシーモードを有効にする。
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}' --type=merge
    ```

1. SMI ポリシーがない場合、「curl」クライアントが「tcp-echo」サービスからの応答を送受信できないことを確認する。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    command terminated with exit code 1
    ```

1. SMI トラフィック アクセスおよびルーティング ポリシーを設定する。
    ```bash
    kubectl apply -f - <<EOF
    # TCP route to allows access to tcp-echo:9000
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: TCPRoute
    metadata:
    name: tcp-echo-route
    namespace: tcp-demo
    spec:
    matches:
        ports:
        - 9000
    ---
    # Traffic target to allow curl app to access tcp-echo service using a TCPRoute
    kind: TrafficTarget
    apiVersion: access.smi-spec.io/v1alpha3
    metadata:
    name: tcp-access
    namespace: tcp-demo
    spec:
    destination:
        kind: ServiceAccount
        name: tcp-echo
        namespace: tcp-demo
    sources:
    - kind: ServiceAccount
        name: curl
        namespace: curl
    rules:
    - kind: TCPRoute
        name: tcp-echo-route
    EOF
    ```

1. `curl` クライアントが SMI TCP ルートを使用して `tcp-echo` サービスからの応答を送受信できることを確認する。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    echo response: hello