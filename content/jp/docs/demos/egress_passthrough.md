---
title: "不明な宛先へのエグレスパススルー"
description: "エグレスポリシーのない外部サービスへのアクセス"
type: ドキュメント
weight: 16
---

このガイドでは、サービスメッシュ内のクライアントが、osm-edge のエグレス機能を使用してメッシュ外部の宛先にアクセスし、エグレスポリシーなしでトラフィックを未知の宛先にパススルーする方法を示す。

## 前提条件
- Kubernetes{{< param min_k8s_version >}}あるいはそれ以上を実行している Kubernetesクラスター。
- osm-edgeがインストールされている。
- APIサーバーとやり取りするためのkubectlは使用可能。
- サービスメッシュを管理するための「osm」CLIは使用可能。


## HTTP(S) メッシュ全体のエグレスパススルーのデモ

1. 有効になっていない場合は、グローバルエグレスパススルーを有効にする。
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge
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

    `curl` クライアントポッドが稼働中であることを確認する。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

1. curl クライアントがポート 443 で httpbin.org Web サイトへの HTTPS リクエストを正常に実行できることを確認する。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I https://httpbin.org:443
    HTTP/2 200
    date: Tue, 16 Mar 2021 22:19:00 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    server: gunicorn/19.9.0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```

    200 OK 応答は、curl クライアントから httpbin.org Web サイトへの HTTPS リクエストが成功したことを示す。

1. メッシュ全体のエグレスが無効になっている場合、HTTPS リクエストが失敗することを確認する。
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
    ```
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I https://httpbin.org:443
	  curl: (7) Failed to connect to httpbin.org port 443 after 3 ms: Connection refused
	  command terminated with exit code 7
    ```
