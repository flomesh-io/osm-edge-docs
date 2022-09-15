---
title: "許容トラフィックポリシーモード"
description: "明示的な SMIポリシーなしでサービスディスカバリを使用してアプリケーション接続を設定する"
type: ドキュメント
weight: 1
---

このガイドでは、明示的な [SMI traffic access policies](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md)を必要とせずに、サービスディスカバリを使用してアプリケーション接続を設定する、osm-edge の 許容なトラフィックポリシー モードを使用して通信する、サービスメッシュ内のクライアントおよびサーバー アプリケーションについて説明する。


## 前提条件
- Kubernetes{{< param min_k8s_version >}}あるいはそれより高いバージョンを実行しているKubernetesクラスター。
- osm-edgeはインストールされている。
-  API サーバーとやり取りするための`kubectl`は使用可能。
- サービスメッシュを管理するための`osm` CLIは使用可能。

## デモ

次のデモは、許容なトラフィックポリシーモードを使用して、HTTP 「curl」クライアントが「httpbin」サービスに HTTP リクエストを送信する様子を示す。

1. 許可モードが有効になっていない場合は有効にする。
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

1. 名前空間をメッシュに登録した後、「httpbin」サービスを「httpbin」名前空間にデプロイする。 `httpbin` サービスはポート `14001` で実行される。

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

    # Deploy httpbin service in the httpbin namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
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
    httpbin-5b8b94b9-lt2vs   2/2     Running   0          20s
    ```

1. その名前空間をメッシュに登録した後、`curl` クライアントを `curl` 名前空間にデプロイする。

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

1. `curl` クライアントがポート `14001` で `httpbin` サービスにアクセスできることを確認する。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://httpbin.httpbin:14001
    HTTP/1.1 200 OK
    server: gunicorn/19.9.0
    date: Wed, 29 Jun 2022 08:50:33 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    access-control-allow-origin: *
    access-control-allow-credentials: true
    connection: keep-alive
    ```

   「200 OK」応答は、「curl」クライアントから「httpbin」サービスへの HTTP リクエストが成功したことを示す。

1. 許容なトラフィックポリシー モードが無効になっている場合、HTTP リクエストが失敗することを確認する。

    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
    ```

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://httpbin.httpbin:14001
    curl: (7) Failed to connect to httpbin.httpbin port 14001: Connection refused
    command terminated with exit code 7
    ```