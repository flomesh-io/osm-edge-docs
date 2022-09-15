---
title: "Cert-manager 証明書プロバイダー"
description: "cert-manager を証明書プロバイダーとして使用する"
type: ドキュメント
weight: 30
---


このガイドでは、証明書プロバイダーとして [cert-manager][1] を使用して、osm-edge で証明書を管理および発行する方法を示す。
## 前提条件
- Kubernetes{{< param min_k8s_version >}}あるいはそれ以上を実行している Kubernetesクラスター。
- APIサーバーとやり取りするためのkubectlは使用可能。
- サービスメッシュをインストール、管理するための「osm」CLIは使用可能。


## デモ

次のデモでは、証明書プロバイダーとして [cert-manager][1] を使用して、osm-edge マネージド サービス メッシュで「相互 TLS (mTLS)」を介して通信する「curl」および「httpbin」アプリケーションに証明書を発行する。

1. `cert-manager` をインストールする。 このデモでは「cert-manager v1.6.1」を使用している。
    ```bash
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
    ```

    ポッドの準備ができており、「cert-manager」名前空間で実行されていることを確認する。

    ```console
    kubectl get pod -n cert-manager
    NAME                                      READY   STATUS    RESTARTS   AGE
    cert-manager-55658cdf68-pdnzg             1/1     Running   0          2m33s
    cert-manager-cainjector-967788869-prtjq   1/1     Running   0          2m33s
    cert-manager-webhook-6668fbb57d-vzm4j     1/1     Running   0          2m33s
    ```

2. osm-edge で証明書を発行できるように、「cert-manager」が必要とする「cert-manager」「Issuer」および「Certificate」リソースを構成する。これらのリソースは、後で osm-edge がインストールされる名前空間に作成する必要がある。
    > 注意: 「cert-manager」を証明書プロバイダーとして使用して「osm-edge」をインストールする前に、issuerの準備が整った状態で「cert-manager」を最初にインストールする必要がある。

    osm-edge がインストールされる名前空間を作成する。

    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl create namespace "$osm_namespace"
    ```

    次に、「SelfSigned」issuerを使用して、カスタムルート証明書をブートストラップする。これにより、「SelfSigned」issuerが作成され、ルート証明書が発行され、メッシュ内のワークロードに発行される証明書の「CA」issuerとしてそのルートを使用する。

    ```bash
    # Create Issuer and Certificate resources
    kubectl apply -f - <<EOF
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: selfsigned
      namespace: "$osm_namespace"
    spec:
      selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: osm-ca
      namespace: "$osm_namespace"
    spec:
      isCA: true
      duration: 87600h # 365 days
      secretName: osm-ca-bundle
      commonName: osm-system
      issuerRef:
        name: selfsigned
        kind: Issuer
        group: cert-manager.io
    ---
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: osm-ca
      namespace: "$osm_namespace"
    spec:
      ca:
        secretName: osm-ca-bundle
    EOF
    ```

3. `osm-ca-bundle` CA シークレットが osm-edge の名前空間で `cert-manager` によって作成されていることを確認する。
   
    ```console
    $ kubectl get secret osm-ca-bundle -n "$osm_namespace"
    NAME            TYPE                DATA   AGE
    osm-ca-bundle   kubernetes.io/tls   3      84s
    ```

    このシークレットに保存された CA 証明書は、インストール時に osm-edge によってその証明書プロバイダー ユーティリティをブートストラップするために使用される。

4. 証明書プロバイダーの種類を `cert-manager` に設定して osm-edge をインストールする。
   
    ```bash
    osm install --set osm.certificateProvider.kind="cert-manager"
    ```

    osm-edge コントロールプレーンポッドの準備ができて実行中であることを確認する。
    
    ```console
    $ kubectl get pod -n "$osm_namespace"
    NAME                              READY   STATUS    RESTARTS   AGE
    osm-bootstrap-7ddc6f9b85-k8ptp    1/1     Running   0          2m52s
    osm-controller-79b777889b-mqk4g   1/1     Running   0          2m52s
    osm-injector-5f96468fb7-p77ps     1/1     Running   0          2m52s
    ```

5. 許容トラフィックポリシーモードを有効にして、自動アプリケーション接続を設定する。
    > 注意: これは「cert-manager」を使用するための要件ではないが、アプリケーション接続に明示的なトラフィックポリシーを必要としないため、デモが簡素化される。

    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

6. 名前空間をメッシュに登録した後、httpbinサービスを httpbin 名前空間にデプロイする。 httpbinサービスはポート14001で実行される。

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

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
    httpbin-5b8b94b9-lt2vs   2/2     Running   0          20s
    ```

7. その名前空間をメッシュに登録した後、`curl` クライアントを `curl` 名前空間にデプロイする。

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

8.  `curl` クライアントがポート `14001` で `httpbin` サービスにアクセスできることを確認する。

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://httpbin.httpbin:14001
    HTTP/1.1 200 OK
    server: gunicorn/19.9.0
    date: Mon, 04 Jul 2022 09:34:11 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    access-control-allow-origin: *
    access-control-allow-credentials: true
    connection: keep-alive
    ```

     200 OK 応答は、curl クライアントから httpbin サービスへの HTTP 要求が成功したことを示す。アプリケーションサイドカー プロキシ間のトラフィックは、cert-manager 証明書プロバイダーによって発行された証明書を活用することにより、相互 TLS (mTLS) を使用して暗号化および認証される。
    


[1]: https://cert-manager.io/