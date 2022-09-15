---
title: "エグレスポリシー"
description: "エグレスポリシーを使用した外部サービスへのアクセス"
type: ドキュメント
weight: 15
---

このガイドでは、サービスメッシュ内のクライアントが、osm-edge の エグレス ポリシー API を使用してメッシュ外部の宛先にアクセスする方法を示す。


## 前提条件
- Kubernetes{{< param min_k8s_version >}}あるいはそれ以上を実行している Kubernetesクラスター。
- osm-edgeがインストールされている。
- APIサーバーとやり取りするためのkubectlは使用可能。
- サービスメッシュを管理するための「osm」CLIは使用可能。


## デモ

1. 有効になっていない場合は、エグレス ポリシーを有効にする。
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
    ```

1. その名前空間をメッシュに登録した後、`curl` クライアントを `curl` 名前空間にデプロイする。
    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    `curl` クライアント ポッドが稼働中であることを確認する。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

## HTTP エグレス

1. `curl` クライアントがポート `80` で `httpbin.org` ウェブサイトへの HTTP リクエスト `http://httpbin.org:80/get` を作成できないことを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    ```

1. `curl` クライアントの サービスアカウントが `http` プロトコルを提供するポート `80` で `httpbin.org` ウェブサイトにアクセスできるようにエグレスポリシーを適用する。
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-80
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 80
        protocol: http
    EOF
    ```

1. `curl` クライアントが `http://httpbin.org:80/get` への HTTP リクエストを成功させることができることを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    HTTP/1.1 200 OK
    date: Mon, 04 Jul 2022 07:48:24 GMT
    content-type: application/json
    content-length: 313
    server: gunicorn/19.9.0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    connection: keep-alive
    ```

1. 上記のポリシーが削除されると、「curl」クライアントが「http://httpbin.org:80/get」への HTTP リクエストを成功させることができなくなることを確認する。
    ```bash
    kubectl delete egress httpbin-80 -n curl
    ```

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    ```

## HTTPS エグレス

HTTPS トラフィックは TLS で暗号化されるため、osm-edge は HTTPS ベースのトラフィックを TCP ストリームとして元の宛先にプロキシすることでルーティングする。 TLS ハンドシェイクで HTTPS クライアントアプリケーションによって示される Server Name Indication (SNI) は、エグレスポリシーで指定されたホストと照合される。

1. `curl` クライアントがポート `443` で `httpbin.org` ウェブサイトへの HTTPS リクエスト `https://httpbin.org:443/get` を作成できないことを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    command terminated with exit code 7
    ```

1. `curl` クライアントの サービスアカウント が `https` プロトコルを提供するポート `443` で `httpbin.org` ウェブサイトにアクセスできるように、エグレスポリシーを適用する。
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-443
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 443
        protocol: https
    EOF
    ```

1. `curl` クライアントが `https://httpbin.org:443/get` への HTTPS リクエストを成功させることができることを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    HTTP/2 200
    date: Thu, 13 May 2021 22:09:36 GMT
    content-type: application/json
    content-length: 260
    server: gunicorn/19.9.0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```

1. 上記のポリシーが削除されると、「curl」クライアントが「https://httpbin.org:443/get」への HTTPS リクエストを成功させることができなくなることを確認する。
    ```bash
    kubectl delete egress httpbin-443 -n curl
    ```
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    command terminated with exit code 7
    ```

## TCP エグレス

TCP ベースのエグレストラフィックは、エグレスポリシーで指定された宛先ポートおよび IP アドレス範囲と照合される。IP アドレス範囲が指定されていない場合、トラフィックは宛先ポートのみに基づいて照合される。 

1. curl クライアントが HTTPS リクエスト https://openservicemesh.io:443 をポート 443 で openservicemesh.io Web サイトに送信できないことを確認する。HTTPS は基礎となるトランスポートプロトコルとして TCP を使用するため、TCP ベースのルーティングは、指定されたポート上の任意の HTTP(s) ホストへのアクセスを暗黙的に有効にする必要がある。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    command terminated with exit code 7
    ```

1. 「curl」クライアントのサービス アカウントが「tcp」プロトコルを提供するポート`443`の任意の宛先にアクセスできるように、エグレスポリシーを適用する。
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: tcp-443
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      ports:
      - number: 443
        protocol: tcp
    EOF
    ```
    > 注意: サーバーがクライアントとサーバー間のデータの最初のバイトを開始する、MySQL、PostgreSQL などのサーバーファーストプロトコルの場合、ポートでプロトコル検出を実行しないように osm-edge に示すには、プロトコルを tcp-server-first に設定する必要がある。プロトコル検出は、接続の最初のバイトの検出に依存しており、これは `server-first` プロトコルとは相容れないものだ。ポートのプロトコルが「tcp-server-first」に設定されている場合、そのポート番号のプロトコル検出はスキップされる。プロトコル検出の実行が必要な他のアプリケーションポートには、「server-first」ポート番号を使用してはならないことに注意することも重要だ。これは、`server-first`プロトコルに使用されるポート番号を、プロトコル検出の実行が必要な「HTTP」や「TCP」などの他のプロトコルで使用してはならないことを意味する。

1. `curl` クライアントが `https://openservicemesh.io:443` への HTTPS リクエストを成功させることができることを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    HTTP/2 200
    cache-control: public, max-age=0, must-revalidate
    content-length: 0
    content-type: text/html; charset=UTF-8
    date: Thu, 13 May 2021 22:35:07 GMT
    etag: "353ebaaf9573718bd1df6b817a472e47-ssl"
    strict-transport-security: max-age=31536000
    age: 0
    server: Netlify
    x-nf-request-id: 35a4f2dc-5356-45dc-9208-63e6fa162e0f-3350874
    ```

1. 上記のポリシーが削除されると、「curl」クライアントが「https://openservicemesh.io:443」への HTTPS リクエストを成功させることができなくなることを確認する。
    ```bash
    kubectl delete egress tcp-443 -n curl
    ```
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    command terminated with exit code 7
    ```

## SMI ルートが一致する HTTPエグレス

HTTPエグレスポリシーでは、HTTPメソッド、ヘッダー、パスに基づいたきめ細かいトラフィック制御のために、SMI HTTPRouteGroupマッチを指定することができる。

1. curl クライアントが http://httpbin.org:80/get および http://httpbin.org:80/status/200 への HTTP リクエストをポート 80 で httpbin.org Web サイトに送信できないことを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    command terminated with exit code 7
    ```

1. SMI HTTPRouteGroup リソースを適用して、HTTP パス `/get` へのアクセス、及びSMI HTTPRouteGroup でマッチするウェブサイトポート `80` の `httpbin.org` へのアクセスを許可する。
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: HTTPRouteGroup
    metadata:
      name: egress-http-route
      namespace: curl
    spec:
      matches:
      - name: get
        pathRegex: /get
    ---
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-80
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 80
        protocol: http
      matches:
      - apiGroup: specs.smi-spec.io/v1alpha4
        kind: HTTPRouteGroup
        name: egress-http-route
    EOF
    ```

1. `curl` クライアントが `http://httpbin.org:80/get` への HTTP リクエストを成功させることができることを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    HTTP/1.1 200 OK
    date: Thu, 13 May 2021 21:49:35 GMT
    content-type: application/json
    content-length: 335
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```

2. `curl` クライアントが `http://httpbin.org:80/status/200` への正常な HTTP リクエストを作成できないことを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    HTTP/1.1 404 Not Found
    date: Fri, 14 May 2021 17:08:48 GMT
    transfer-encoding: chunked
    ```

3. 一致する SMI HTTPRouteGroup リソースを更新して、正規表現「/status.*」に一致する HTTP パスへのリクエストを許可する。
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: HTTPRouteGroup
    metadata:
      name: egress-http-route
      namespace: curl
    spec:
      matches:
      - name: get
        pathRegex: /get
      - name: status
        pathRegex: /status.*
    EOF
    ```

4. `curl` クライアントが `http://httpbin.org:80/status/200` への HTTP リクエストを正常に行えるようになったことを確認する。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    HTTP/1.1 200 OK
    date: Fri, 14 May 2021 17:10:48 GMT
    content-type: text/html; charset=utf-8
    content-length: 0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```
