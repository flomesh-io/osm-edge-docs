---
title: "快速体验"
description: "5 分钟快速体验 osm-ege"
type: docs
weight: 2
---

# osm-edge 快速体验

以下演示如何在 5 分钟之内，下载、安装、运行 osm-edge，并部署一个演示应用，并完成链路加密、访问控制、流量分割等 SMI 标准功能。该演示使用 x86 版本的 Ubuntu 21，运行`v1.23.8+k3s1`版本的 k3s。更多版本和平台的支持，请参考完整的[新手上路文档](/docs/getting_started/)。

## 先决条件

一个运行中的 Kubernetes 集群，可以通过下面的命令快速创建 k3s 单节点集群：

```bash
export INSTALL_K3S_VERSION=v1.23.8+k3s1
curl -sfL https://get.k3s.io | sh -s - --disable traefik --write-kubeconfig-mode 644 --write-kubeconfig ~/.kube/config
```

> osm-edge支持的最低 Kubernetes 版本为1.20。

## 下载并安装 osm-edge 命令行工具

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release={{< param osm_edge_version >}}
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

## 在 Kubernetes 上安装 osm-edge

> 此命令启用 [Prometheus](https://github.com/prometheus/prometheus)、[Grafana](https://github.com/grafana/grafana) 和 [Jaeger](https://github.com/jaegertracing/jaeger) 集成

```bash
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.deployPrometheus=true \
    --set=osm.deployGrafana=true \
    --set=osm.deployJaeger=true
```
## 部署演示应用

演示应用包括了如下服务：

- `bookbuyer` 是一个 HTTP 客户端，它发送请求给 `bookstore`。这个流量是**允许**的。
- `bookthief` 是一个 HTTP 客户端，很像 `bookbuyer`，也会发送 HTTP 请求给 `bookstore`。这个流量应该被**阻止**。
- `bookstore` 是一个服务器，负责对 HTTP 请求给与响应。同时，该服务器也是一个客户端，发送请求给 `bookwarehouse` 服务。这个流量是被**允许**的。
- `bookwarehouse` 是一个服务器，应该只对 `bookstore` 做出响应。`bookbuyer` 和 `bookthief` 都应该被其阻止。
- `mysql` 是一个 MySQL 数据库，只有 `bookwarehouse` 可以访问。

使用如下命令部署这些服务：

```bash
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
osm namespace add bookstore bookbuyer bookthief bookwarehouse
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookbuyer.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookthief.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookstore.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookwarehouse.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/mysql.yaml
```

把每个服务的GUI端口对外暴露，这样用浏览器我们可以访问这些端口，观察演示的现象。

```bash
git clone https://github.com/flomesh-io/osm-edge.git -b {{< param osm_branch >}}
cd osm-edge
cp .env.example .env
./scripts/port-forward-all.sh #可以忽略错误信息
```

在一个浏览器中，打开下面的 URL：

_注意：如果需要从宿主机访问，需要将 `localhost` 替换成虚拟机的 IP 地址；或者在宿主机上运行 `port-forward-all.sh` 脚本。_

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**
- [http://localhost:8084](http://localhost:8084) - **bookstore**


## 访问控制

通过上面的命令安装 osm-edge，所有的服务都是没有访问控制的（宽松流量模式），或者说所有的访问都是允许的。通过在浏览器中观察每个服务的页面数量增长可以看到没有访问控制时候的情况：

在 `bookbuyer`、`bookthief` UI 中的计数分别对应了购买和盗窃的书籍数量，而在 `bookstore-v1` 中这些都应该在增加：

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**

在 `bookstore` UI 中的对于书籍销售的计数也应该在增加：

- [http://localhost:8084](http://localhost:8084) - **bookstore**

接下来演示通过禁用宽松流量策略模式，拒绝对 `bookstore` 服务的访问：

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

此时会发现计数将不再增加。

现在执行下面的命令，放行 `bookbuyer` 对 `bookstore` 的访问：

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/access/traffic-access-v1.yaml
```

这里再去查看 `bookbuyer` 和 `bookstore` UI，会发现计数恢复增加，而 `bookthief` UI 的计数仍然停止。

通过访问控制，我们成功阻止 `bookthief` 从 `bookstore` 盗窃书籍，而正常的购买不受影响。

