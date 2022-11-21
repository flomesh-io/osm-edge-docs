---
title: "配置流量拆分"
description: "使用 SMI Traffic Split API 来平衡服务之间的流量"
type: docs
weight: 4
---

# 配置两个服务之间的流量拆分

我们将演示如何均衡两个 Kubernetes 服务之间的流量，通常叫做流量拆分。我们将在后端 `bookstore` 服务和 `bookstore-v2` 服务之间来拆分原来定向到根 `bookstore` 服务的流量。`bookstore` 服务和 `bookstore-v2` 服务也被看做叶子服务。了解更多关于如何为流量拆分配置服务请参阅[流量拆分 How-To 指南](/docs/guides/traffic_management/traffic_split.md)。

### 部署 bookstore v2 应用

要演示 SMI 流量访问和拆分策略的使用，我们现在将要部署 bookstore 应用的 v2 版本（`bookstore-v2`）。

```bash
# Contains the bookstore-v2 Kubernetes Service, Service Account, Deployment and SMI Traffic Target resource to allow
# `bookbuyer` to communicate with `bookstore-v2` pods
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookstore-v2.yaml
```

等待 `bookstore-v2` pod 在 `bookstore` 命名空间里运行起来，退出并重新启动 port-forward 脚本以可以访问 bookstore-v2。

```bash
bash <<EOF
./scripts/port-forward-bookbuyer-ui.sh &
./scripts/port-forward-bookstore-ui.sh &
./scripts/port-forward-bookstore-ui-v2.sh &
./scripts/port-forward-bookthief-ui.sh &
wait
EOF
```

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

`bookstore-v2` 计数器应该增长，因为到根 `bookstore` 服务的流量是通过标签选择器配置的，该选择器选择了包括 `bookstore-v1` 和 `bookstore-v2` 后端 pod 在内的端点。

### 创建 SMI Traffic Split

部署 SMI 流量拆分策略来定向发送到根 `bookstore` 服务 100% 的流量到 `bookstore` 服务后端。这一点是必须的，以确保定向到 `bookstore` 的流量只被定向到 bookstore 应用的 `version v1`，其包括了支持 `bookstore-v1` 服务的 pod。`TrafficSplit` 配置随后将被更新为定向一部分百分比的流量给使用 `bookstore-v2` 叶服务的 bookstore 服务 `version v2`

基于这个原因，很重要的一点是如果需要流量拆分，就要确保客户端应用一直保持和根服务通信。如若不然，当流量拆分被需要时，客户端应用就要更新为和根服务通信。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v1.yaml
```

_注意：根服务是一个 Kubernetes 服务，它的选择器需要匹配支持叶服务的 pod。在这个演示中，根服务 `bookstore` 有选择器 `app:bookstore`，其分别匹配了在 `bookstore (v1)` 和 `bookstore-v2` 部署上的标签 `app:bookstore,version:v1` 和 `app:bookstore,version=v2`。该根服务能够被按照带或者不带 `.<namespace>.svc.cluster.local` 后缀的服务名称来引用 SMI 流量拆分中的资源。_

对于书籍销售的计数，从 `bookstore-v2` 浏览器窗口看应该停止增长。这是因为当前的流量拆分策略把权重 100 给了 `bookstore-v1` 而排除了支持 `bookstore-v2` 服务的 pod。可以通过运行下面命令并同时观察**后端**的属性来验证流量拆分策略。

```bash
kubectl describe trafficsplit bookstore-split -n bookstore
```

### 拆分流量给 bookstore-v2

更新 SMI 流量拆分策略来定向 50% 的流量发送到根 `bookstore` 服务再到 `bookstore` 服务，然后 50% 的流量到 `bookstore-v2` 服务，这可以通过添加 `bookstore-v2` 后端到规范并修改权重域来完成。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/split/traffic-split-50-50.yaml
```

等待变更的传播，并且在浏览器窗口里观察 `bookstore` 和 `bookstore-v2` 计数器增长。两个计数器都应该增长：


- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

### 拆分全部流量到 bookstore-v2

更新 `bookstore-split` 流量拆分来配置全部的流量去往 `bookstore-v2`：

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v2.yaml
```

等待变更的下发并在浏览器窗口里观察 `bookstore-v2` 计数器的增长和 `bookstore` 计数器的停止：

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
- [http://localhost:8083](http://localhost:8084) - **bookstore**

现在，定向到 `bookstore` 的全部流量都流向了 `bookstore-v2`。

## 下一步

- [用 Prometheus 和 Grafana 配置可观测性](/docs/getting_started/observability/)
- [清理示例应用并卸载 osm-edge](/docs/getting_started/cleanup/)