---
title: "FSM 入口"
description: "FSM 入口控制器实现的 HTTP 入口"
type: docs
weight: 10
draft: false
---

osm-edge 可以选择使用 [FSM](git@github.com:flomesh-io/fsm.git) 入口控制器和基于 Pipy 的边缘代理来路由外部的流量到服务网格后端。这个指南演示了如何为 osm-edge 服务网格管理的服务配置 HTTP ingress。

## 先决条件 pipy

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 使用 `kubectl` 与 API server 交互。
- 未安装 osm-edge。如果已安装必须先删除。
- 已安装 `osm` 或者 `Helm 3` 命令行工具，用于安装 osm-edge 和 FSM。
- osm-edge 版本 >= v1.1.0。


## 演示

首先，在 `osm-system` 命名空间下安装 osm-edge 和 Contour，并将网格名字命名为 `osm`。
```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge will be installed
export osm_mesh_name=osm # Replace osm with the desired osm-edge mesh name
```

使用 `osm` 命令行工具：
```bash
osm install --set fsm.enabled=true \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" 
```

使用 `Helm` 安装：
```bash
helm install "$osm_mesh_name" osm --repo https://flomesh-io.github.io/osm-edge \
    --set fsm.enabled=true
```

为了通过对访问后端流量的限制来对客户端授权，我们将配置 IngressBackend，这样只有来自 `ingress-pipy-controller` 端点的入口流量才可以路由到后端服务。为了发现 `ingress-pipy-controller` 端点，我们需要 osm-edge 控制器及监视相应的命名空间。但是为了保证 FSM 功能正常，不能为其注入 Pipy sidecar。

```bash
kubectl label namespace "$osm_namespace" openservicemesh.io/monitored-by="$osm_mesh_name"
```

保存入口网关的外部 IP 地址和端口，后面会用其测试访问后端应用。

```bash
export ingress_host="$(kubectl -n "$osm_namespace" get service ingress-pipy-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
export ingress_port="$(kubectl -n "$osm_namespace" get service ingress-pipy-controller -o jsonpath='{.spec.ports[?(@.name=="http")].port}')"
```

下一步是部署示例 `httpbin` 服务。

```bash
# Create a namespace
kubectl create ns httpbin

# Add the namespace to the mesh
osm namespace add httpbin

# Deploy the application
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

确保 `httpbin` 服务和 pod 启动并正常运行：

```console
$ kubectl get pods -n httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-74677b7df7-zzlm2   2/2     Running   0          11h

$ kubectl get svc -n httpbin
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
httpbin   ClusterIP   10.0.22.196   <none>        14001/TCP   11h
```

### HTTP Ingress

接下来，创建必要的 HTTPProxy 和 IngressBackend 配置来允许外部客户端访问 `httpbin` 命名空间下 `httpbin` 服务的 `14001` 端口。因为没有使用 TLS，Contour入口网关到 `httpbin` 后端 pod 的链接没有加密。

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
    http:
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

现在，我们期望外部客户端可以访问 `httpbin` 服务，HTTP 请求的 `HOST` 请求头为 `httpbin.org`：

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