---
title: "入口"
description: "使用入口管理对群集内服务的外部访问"
weight: 5
---

# 入口

## 使用入口管理集群内服务的外部访问

入口是指管理对集群内服务的外部访问，通常是 HTTP/HTTPS 服务。osm-edge 的入口功能允许集群管理员和应用程序所有者使用一组规则将流量从服务网格外部的客户端路由到服务网格后端，具体取决于用于执行入口的机制。

## IngressBackend API

osm-edge 利用其 [IngressBackend API][1] 来配置后端服务以接受来自可信来源的入口流量。该规范允许配置特定后端如何根据使用的协议（HTTP 或 HTTPS）授权入口流量。当后端协议为 `http` 时，指定的源类型必须是： 1. `Service` 类型，其端点将被授权连接到后端，或 2. `IPRange` 类型，指定授权的源 IP CIDR 范围连接到后端。当后端协议是 `https` 时，指定的源必须是 `AuthenticatedPrincipal` 类型，它定义了后端将验证的客户端证书中编码的主题备用名称（SAN）。`Service` 或 `IPRange` 类型的源对于 `https` 后端是可选的，如果指定，则意味着客户端必须匹配源以及其 `AuthenticatedPrincipal` 值。对于 `https` 后端，默认执行客户端证书验证，可以通过在后端的 `tls` 字段中设置 `skipClientCertValidation: true` 来禁用。`IngressBackend` 配置中 `backend` 服务的 `port.number` 字段必须与Kubernetes 服务的 `targetPort` 对应。

请注意，当 `IngressBackend` 配置中源的 `Kind` 设置为 `Service` 时，osm-edge 控制器将尝试发现该服务的端点。为了使 osm-edge 能够发现服务的端点，服务所在的命名空间需要是受监控的命名空间。使用以下命令启用要监视的命名空间：

```bash
kubectl label ns <namespace> openservicemesh.io/monitored-by=<mesh name>
```

### 示例

下面的 IngressBackend 配置将使 `test` 命名空间下的 `foo` service 的 `80` 端口只允许被位于 `default` 命名空间下的 `myapp` service 的端点访问。

```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http
  sources:
    - kind: Service
      namespace: default
      name: myapp
```

下面的 IngressBackend 配置将使 `test` 命名空间下的 `foo` service 的 `80` 端口只允许被属于 CIDR 范围 `10.0.0.0/8` 的源 IP 访问。

```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http
  sources:
    - kind: IPRange
      name: 10.0.0.0/8
```

下面的 IngressBackend 配置将使 `test` 命名空间下的 `foo` service 的 `80` 端口只允许被使用 `TLS` 加密且客户端证书中编码的主题备用名（SAN）为 `client.default.svc.cluster.local` 的客户端访问。

```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80
        protocol: https # https implies TLS
      tls:
        skipClientCertValidation: false # mTLS (optional, default: false)
  sources:
    - kind: AuthenticatedPrincipal
      name: client.default.svc.cluster.local
```

参阅下面的部分以了解如何配置 `http` 和 `https` 后端的 `IngressBackend`。

## 执行入口的选择

osm-edge 支持多种选项来使用 ingress 从外部公开网格服务，这些将在以下部分中描述。 osm-edge 已经通过 [FSM（Flomesh Service Mesh）](https://github.com/flomesh-io/fsm) 和 OSS Nginx 进行了测试，它们与安装在网格外并提供证书以参与网格的入口控制器一起工作。

> 注意：osm-edge 与 Nginx Plus 的集成尚未经过全面测试，可以从 Kubernetes secret 中获取自签名 mTLS 证书。但是，集成 Nginx Plus 或任何入口的另一种方法是将其安装在网格中，以便将为其注入 Pipy sidecar，这将允许它参与到网格中。另外其他入站端口（例如 80 和 443）可能需要被允许绕过 Pipy sidecar。

### 1. 使用 FSM 入口控制器和网关

使用 [FSM](https://github.com/flomesh-io/fsm) 入口控制器和边缘代理是在 osm-edge 托管服务网格中执行 Ingress 的首选方法。使用 FSM，用户可以获得具有丰富策略规范的高性能入口控制器，适用于各种场景，同时保持轻量级配置文件。

将 FSM 用作入口，在安装网格时启用它：

```bash
osm install --set fsm.enabled=true
```

除了使用适当的 API 配置 FSM 的边缘代理之外，osm-edge 中的服务网格后端将只接受来自授权边缘代理或网关的流量。osm-edge 的 [IngressBackend 规范][1] 允许集群管理员和应用程序所有者明确指定服务网格后端应如何授权入口流量。以下部分描述了如何结合使用 `IngressBackend` 和 `HTTPProxy` API，以允许将 HTTP 和 HTTPS 入口流量路由到网格后端。

建议始终将入口流量限制为授权客户端。为此，使 osm-edge 能够监控位于 osm-edge 安装所在的命名空间中的 FSM 边缘代理的端点：

```bash
kubectl label ns <osm namespace> openservicemesh.io/monitored-by=<mesh name>
```

#### 使用 FSM 的 HTTP Ingress

一个最小的 [HTTPProxy][2] 配置和 osm-edge 的 `IngressBackend`[1] 规范将入口流量路由到命名空间 `test` 中的网格服务 `foo` 可能如下所示：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fsm-ingress
  namespace: test
spec:
  ingressClassName: pipy
  rules:
  - host: foo-basic.bar.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: foo
            port:
              number: 80
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http # http implies no TLS
  sources:
    - kind: Service
      namespace: osm-system
      name: ingress-pipy-controller
```

上面的配置允许外部客户端访问 `test` 命名空间下的 `foo` service：

1. Ingress 配置会将来自外部的带  `foo-basic.bar.com` 的 `Host:` 标头的传入 HTTP 流量路由到 `test` 命名空间中端口 `80` 上名为 `foo` 的服务。
2. IngressBackend 配置只允许来自同在安装 osm-edge 的命名空间（默认为 `osm-system`）下名为 `ingress-pipy-controller` service 的端点访问 `test` 命名空间下的 `foo` serivce 的 `80` 端口。

#### 示例

参阅 [FSM 入口示例](/docs/demos/ingress_fsm) 作为示例来了解如何在 osm-edge 使用 FSM 将网格服务队外暴露。

### 2. 自带入口控制器和网关

如果将 osm-edge 与 FMS 一起用于入口不适合你的用例，osm-edge 提供了使用你自己的入口控制器和边缘网关将外部流量路由到服务网格后端的工具。与上面的入口配置方式非常相似，除了配置入口控制器以将流量路由到服务网格后端之外，还需要 IngressBackend 配置来授权负责代理来自外部流量的客户端。

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.IngressBackendSpec
[2]: https://projectcontour.io/docs/v1.18.0/config/fundamentals/
