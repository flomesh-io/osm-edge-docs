---
title: "Traefikを使用したイングレス"
description: "Traefikイングレスコントローラーを使用したKubernetesイングレス"
type: ドキュメント
weight: 12
draft: false
---

この記事では、[Traefik Ingress](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)を使用して、osm-edge サービスメッシュによってホストされているサービスにアクセスする方法を示す。

## 前提条件 

* Kubernetesクラスターのバジョンは v1.19.0 あるいはもっと高い。
* `kubectl` を使用して API サーバーとやり取りする。
* osm-edge はインストールされていない。インストールされている場合はまず削除する必要がある。
* osm-edge をインストールするには、osm cli をインストールする。
* traefikインストール用に Helm 3 コマンド ラインツールがインストールされる。
* osm-edge バージョン >= v1.1.0.

## デモ

### Traefikをインストールする

```shell
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install traefik traefik/traefik -n traefik --create-namespace
```

ポッドが稼働中であることを確認する。

```shell
kubectl get po -n traefik
NAME READY STATUS RESTARTS AGE
traefik-69fb598d54-9v9vf 1/1 Running 0 24s
```

エントリゲートウェイの外部IPアドレスとポートを取得して環境変数に保存する。これは、後でアプリケーションにアクセスするために使用される。

```
export ingress_host="$(kubectl -n traefik get service traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
export ingress_port="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[? (@.name=="web")].port}')"
```

### osm-edgeをインストールする

```shell
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true
```

ポッドが稼働中であることを確認する。

```shell
kubectl get po -n osm-system
NAME READY STATUS RESTARTS AGE
osm-bootstrap-6477f776cc-d5r89 1/1 Running 0 2m51s
osm-injector-5696694cf6-7kvpt 1/1 Running 0 2m51s
osm-controller-86d68c557b-tvgtm 2/2 Running 0 2m51s
```

### サンプル サービスをデプロイする

```shell
kubectl create ns httpbin
osm namespace add httpbin
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

サービスが作成され、ポッドが稼働中であることを確認する。
```shell
kubectl get svc -n httpbin
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
httpbin ClusterIP 10.43.51.114 <none> 14001/TCP 9s

kubectl get po -n httpbin
NAME READY STATUS RESTARTS AGE
httpbin-69dc7d545c-bsjxx 2/2 Running 0 77s
```

### HTTP イングレス

次に、`httpbin` 名前空間で `httpbin` サービスの `14001` ポートを公開するイングレスを作成する。

```shell
kubectl apply -f - <<EOF
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: httpbin
  namespace: httpbin
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - host: httpbin.org
     http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
EOF
```

前に保存されたエントリ ゲートウェイのアドレスとポートを使用してサービスにアクセスすると、この時点で「502」の応答を受け取るはずだ。エントリゲートウェイが `httpbin` サービスにアクセスできるようにするために `IngressBackend` を作成する必要があるため、これは正常だ。

```shell
curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 502 Bad Gateway
Date: Tue, 09 Aug 2022 13:17:11 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8
```

以下のコマンドを実行して IngressBackend を作成する。

```shell
kubectl apply -f - <<EOF
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
    namespace: traefik
    name: traefik
EOF
```

ここで、再度 httpbin にアクセスすると、正常にアクセスできるようになる。

```shell
curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Content-Length: 338
Content-Type: application/json
Date: Tue, 09 Aug 2022 13:17:41 GMT
Osm-Stats-Kind: Deployment
Osm-Stats-Name: httpbin
Osm-Stats-Namespace: httpbin
Osm-Stats-Pod: httpbin-69dc7d545c-bsjxx
Server: gunicorn/19.9.0
```