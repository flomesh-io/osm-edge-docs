---
title: "FSM付きのイングレス"
description: "FSM イングレス コントローラーによって実装された HTTP イングレス"
type: ドキュメント
weight: 10
draft: false
---

 osm-edge は、オプションで[FSM](git@github.com:flomesh-io/fsm.git) イングレス コントローラーと Pipy ベースのエッジ プロキシを使用して、外部トラフィックをサービス メッシュ バックエンドにルーティングできる。このガイドでは、osm-edge サービス メッシュによって管理されるサービスの HTTP イングレスを構成する方法を示す。

## 前提条件

- Kubernetesクラスターバージョン {{< param min_k8s_version >}}あるいはそれより高いバージョン。
- `kubectl` を使用して API サーバーとやり取りする。
- osm-edge はインストールされていない。インストールされている場合は削除する必要がある。
- osm-edge および FSM をインストールするための「osm」または「Helm 3」コマンド ライン ツールをインストールした。
- osm-edge バージョン >= v1.1.0.

## デモ

まず、「osm-system」名前空間に osm-edge と fsm をインストールし、グリッドに「osm」という名前を付ける。

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge will be installed
export osm_mesh_name=osm # Replace osm with the desired osm-edge mesh name
```

`osm` コマンドラインツールを使用する。

```bash
osm install --set fsm.enabled=true \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace"
```

``Helm`` を使用してインストールする。

```bash
helm install "$osm_mesh_name" osm --repo https://flomesh-io.github.io/osm-edge \
    --set fsm.enabled=true
```

バックエンドトラフィックへのアクセスを制限してクライアントを認証するために、`ingress-pipy-controller`エンドポイントからのイングレストラフィックのみがバックエンドサービスにルーティングできるようにIngressBackendを設定する。 ingress-pipy-controller`のエンドポイントを発見するためには、osm-edgeコントローラと、それを監視するための対応する名前空間が必要だ。ただし、FSM が正常に機能することを保証するために、Pipy サイドカーを挿入することはできない。

```bash
kubectl label namespace "$osm_namespace" openservicemesh.io/monitored-by="$osm_mesh_name"
```

後でバックエンド アプリケーションへのアクセスをテストするために使用する、エントリゲートウェイの外部 IP アドレスとポートを保存する。

```bash
export ingress_host="$(kubectl -n "$osm_namespace" get service ingress-pipy-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}') "
export ingress_port="$(kubectl -n "$osm_namespace" get service ingress-pipy-controller -o jsonpath='{.spec.ports[? (@.name=="http")].port}')"
```

次のステップは、サンプルの「httpbin」サービスをデプロイすることだ。

```bash
# Create a namespace
kubectl create ns httpbin

# Add the namespace to the mesh
osm namespace add httpbin

# Deploy the application
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin. yaml -n httpbin
```

`httpbin` サービスとポッドが正常に稼働していることを確認する。

```console
$ kubectl get pods -n httpbin
NAME READY STATUS RESTARTS AGE
httpbin-74677b7df7-zzlm2 2/2 Running 0 11h

$ kubectl get svc -n httpbin
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
httpbin ClusterIP 10.0.22.196 <none> 14001/TCP 11h
```

### HTTP イングレス

次に、必要な HTTPProxy および IngressBackend 設定を作成して、外部クライアントが「httpbin」名前空間の「httpbin」サービスのポート「14001」にアクセスできるようにする。TLS が使用されていないため、fsm エントリ ゲートウェイから「httpbin」バックエンド ポッドへのリンクは暗号化されていない。

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
spec:
  ingressClassName: pipy
  rules:
  - host: httpbin.org
    paths: http:
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
    namespace: "$osm_namespace"
    name: ingress-pipy-controller
EOF
```

ここで、HTTPリクエストの `HOST` リクエストヘッダが `httpbin.org` で、外部クライアントが `httpbin` サービスにアクセスすることを想定している。

```console
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
server: gunicorn/19.9.0
date: Tue, 05 Jul 2022 07:34:11 GMT
content-type: application/json
content-length: 241
access-control-allow-origin: *
access-control-allow-credentials: true
connection: keep-alive
```