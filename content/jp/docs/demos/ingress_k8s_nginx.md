---
title: "Kubernetes Nginx イングレス コントローラーを使用したイングレス"
description: "Kubernetes Nginx イングレス コントローラーを使用した HTTP および HTTPS イングレス"
type: ドキュメント
weight: 11
---

このガイドでは、 [Kubernetes Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)を使用する場合に、osm-edge マネージド サービス メッシュのサービス部分への HTTP および HTTPS イングレスを設定する方法を示す。


## 前提条件
- Kubernetes {{< param min_k8s_version >}}あるいはそれ以上を実行しているKubernetesクラスター。
-  API サーバーとやり取りするための`kubectl`は使用可能。
- インストールされているosm-edge バージョン >= v1.1.0.
- Kubernetes Nginx Ingress Controllerがインストールされていること。[deployment guide](https://kubernetes.github.io/ingress-nginx/deploy/)を参考にインストールしてください。
## デモ
まず、osm-edge と Nginx のインストールに関する詳細に注意してください。

```bash
osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
osm_mesh_name=osm # replace osm with the mesh name (use `osm mesh list` command)

nginx_ingress_namespace=<nginx-namespace> # replace <nginx-namespace> with the namespace where Nginx is installed
nginx_ingress_service=<nginx-ingress-controller-service> # replace <nginx-ingress-controller-service> with the name of the nginx ingress controller service
nginx_ingress_host="$(kubectl -n "$nginx_ingress_namespace" get service "$nginx_ingress_service" -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
nginx_ingress_port="$(kubectl -n "$nginx_ingress_namespace" get service "$nginx_ingress_service" -o jsonpath='{.spec.ports[?(@.name=="http")].port}')"
```

バックエンドのイングレス トラフィックを許可されたクライアントに制限するには、Nginx Ingress Controller サービスのエンドポイントからのイングレス トラフィックのみがトラフィックをサービス バックエンドにルーティングできるように、IngressBackend 構成をセットアップする。このサービスのエンドポイントを発見できるようにするには、対応する名前空間を監視するための osm-edge コントローラーが必要だ。ただし、Nginxが正しく動作するためには、Pipyのサイドカーを挿入してはならない。

```bash
osm namespace add "$nginx_ingress_namespace" --mesh-name "$osm_mesh_name" --disable-sidecar-injection
```

次に、サンプルの `httpbin` サービスをデプロイする。

```bash
# Create a namespace
kubectl create ns httpbin

# Add the namespace to the mesh
osm namespace add httpbin

# Deploy the application
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

「httpbin」サービスとポッドが稼働中であることを確認する。

```console
$ kubectl get pods -n httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-74677b7df7-zzlm2   2/2     Running   0          11h

$ kubectl get svc -n httpbin
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
httpbin   ClusterIP   10.0.22.196   <none>        14001/TCP   11h
```

### HTTP イングレス

次に、外部クライアントが httpbin 名前空間のポート 14001 で httpbin サービスにアクセスできるようにするために必要な Ingress および IngressBackend 設定を作成する。TLS を使用していないため、Nginx のイングレス サービスから「httpbin」バックエンド ポッドへの接続は暗号化されていない。

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: http
  sources:
  - kind: Service
    namespace: "$nginx_ingress_namespace"
    name: "$nginx_ingress_service"
EOF
```

ここで、外部クライアントが HTTP リクエストのために `httpbin` サービスにアクセスできることを想定している:

```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Mon, 04 Jul 2022 06:55:26 GMT
Content-Type: application/json
Content-Length: 346
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
```

### HTTPS イングレス (mTLS and TLS)

HTTPS バックエンドへの接続をプロキシするために、Ingress と IngressBackend の設定でバックエンドプロトコルとして https を使用し、Nginx が TLS バックエンドへの HTTPS 接続をプロキシするクライアント証明書として osm-edge が証明書を発行するように設定することにする。 クライアント証明書とCA証明書は、Nginxがサービスメッシュバックエンドを認証するために使用するKubernetesシークレットに保存される。Nginxのイングレスサービスのクライアント証明書を発行するには、`osm-mesh-config` `MeshConfig` リソースを更新する。

```bash
kubectl edit meshconfig osm-mesh-config -n "$osm_namespace"
```

spec.certificate`の下に`ingressGateway`フィールドを追加する。

```yaml
certificate:
  ingressGateway:
    secret:
      name: osm-nginx-client-cert
      namespace: <osm-namespace> # replace <osm-namespace> with the namespace where osm-edge is installed
    subjectAltNames:
    - ingress-nginx.ingress-nginx.cluster.local
    validityDuration: 24h
```
> 注意: Subject Alternative Name (SAN) の形式は`<service-account>.<namespace>.cluster.local` で、サービス アカウントと名前空間は Ngnix サービスに対応する。

次に、mTLS を介したバックエンドへのプロキシを有効にしながら、バックエンド サービスへの TLS プロキシを使用する Ingress および IngressBackend 設定を作成する必要がある。これを機能させるには、httpbin` サービスに向けられた HTTPS イングレス・トラフィックが、信頼できるクライアントからのトラフィックのみを受け入れるように指定する IngressBackend リソースを作成する必要がある。osm-edge は、Nginx ingress サービスのクライアント証明書を Subject ALternative Name (SAN) `ingress-nginx.cluster.local` でプロビジョニングしたため、IngressBackendの設定は、Nginxのイングレスサービスとhttpbin`バックエンドの間でmTLS認証のために同じSANを参照する必要がある。

設定を適用する

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "httpbin.httpbin.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/osm-nginx-client-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
---
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```

ここで、外部クライアントが、イングレス ゲートウェイとサービスバックエンドの間で mTLS を介した HTTPS プロキシを使用して、リクエストの「httpbin」サービスにアクセスできることを想定している。

```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Mon, 04 Jul 2022 06:55:26 GMT
Content-Type: application/json
Content-Length: 346
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
```

承認されていないクライアントがバックエンドへのアクセスを許可されていないことを確認するには、IngressBackend 設定で指定された「ソース」を更新する。プリンシパルを、Nginx クライアントの証明書でエンコードされた SAN 以外のものに更新しましょう。プリンシパルを、Nginxクライアントの証明書にエンコードされているSAN以外のものに更新してみよう。

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: untrusted-client.cluster.local # untrusted
EOF
```

「HTTP 403 Forbidden」レスポンスでリクエストが拒否されることを確認する。

```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 403 Forbidden
Date: Wed, 18 Aug 2021 18:36:09 GMT
Content-Type: text/plain
Content-Length: 19
Connection: keep-alive
```

次に、信頼できないクライアントを引き続き使用しながら、IngressBackend 設定を更新して「skipClientCertValidation: true」を設定することにより、必要に応じてサービス バックエンドでクライアント証明書の検証を無効にするサポートを示す。

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: true
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: untrusted-client.cluster.local # untrusted
EOF
```

信頼できない認証済みプリンシパルがバックエンドへの接続を許可されているため、リクエストが再び成功することを確認する。

```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Mon, 04 Jul 2022 06:55:26 GMT
Content-Type: application/json
Content-Length: 346
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
```
