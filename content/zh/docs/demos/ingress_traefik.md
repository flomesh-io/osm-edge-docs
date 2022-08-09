---
title: "Traefik Ingress 入口"
description: "使用 Traefik Ingress 入口控制器的入口"
type: docs
weight: 12
draft: false
---

本文将演示如何使用 [Traefik Ingress](https://doc.traefik.io/traefik/providers/kubernetes-ingress/) 来访问 osm-edge 服务网格托管的服务。

## 先决条件 

* Kubernetes 集群版本 v1.19.0 或者更高。
* 使用 kubectl 与 API server 交互。
* 未安装 osm-edge，如果已安装必须先删除。
* 已安装 osm cli 用于安装 osm-edge。
* 已安装 Helm 3 命令行工具，用于安装 traefik。
* osm-edge 版本 >= v1.1.0。

## Demo

### 安装 Traefik

```shell
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install traefik traefik/traefik -n traefik --create-namespace
```

确认 pod 已经启动并运行。

```shell
kubectl get po -n traefik
NAME                       READY   STATUS    RESTARTS   AGE
traefik-69fb598d54-9v9vf   1/1     Running   0          24s
```

保存入口网关的外部 IP 地址和端口，后面会用其访问应用。

```
export ingress_host="$(kubectl -n traefik get service traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
export ingress_port="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="web")].port}')"
```

### 安装 osm

```shell
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true
```

确认 pod 已经启动并运行。

```shell
kubectl get po -n osm-system
NAME                              READY   STATUS    RESTARTS   AGE
osm-bootstrap-6477f776cc-d5r89    1/1     Running   0          2m51s
osm-injector-5696694cf6-7kvpt     1/1     Running   0          2m51s
osm-controller-86d68c557b-tvgtm   2/2     Running   0          2m51s
```

### 部署示例服务

```shell
kubectl create ns httpbin
osm namespace add httpbin
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

确认已创建 service，pod 已启动并运行。

```shell
kubectl get svc -n httpbin
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
httpbin   ClusterIP   10.43.51.114   <none>        14001/TCP   9s

kubectl get po -n httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-69dc7d545c-bsjxx   2/2     Running   0          77s
```

### HTTP Ingress

接下来，创建 ingress 来暴露 `httpbin` 命名空间下 `httpbin` service 的 `14001` 端口。

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

使用前面保存的入口网关地址和端口来访问服务，此时应该会收到 `502` 的响应。这是正常的，因为此时还需要创建 `IngressBackend` 来允许入口网关访问 `httpbin` service。

```shell
curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 502 Bad Gateway
Date: Tue, 09 Aug 2022 13:17:11 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8
```

执行下面的命令来创建  `IngressBackend`。

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

现在，重新访问 `httpbin` 就可以成功访问了。

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