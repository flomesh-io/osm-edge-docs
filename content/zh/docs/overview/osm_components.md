---
title: "osm-edge 控制平面组件"
description: "组件"
type: docs
weight: 2
---

## 检查 osm-edge 组件

一些 osm-edge 组件将被默认安装在选中的命名空间下，默认是 `osm-system`。要检查它们可以通过如下的 `kubectl` 命令：

```console
# Replace osm-system with the namespace where osm-edge is installed
$ kubectl get pods,svc,secrets,meshconfigs,serviceaccount --namespace osm-system
```

一些集群范围（非命名空间的）osm-edge 组件也将被安装。检查它们可以使用如下的 `kubectl` 命令：

```console
kubectl get clusterrolebinding,clusterrole,mutatingwebhookconfiguration
```

在底层，`osm` 在控制平面所在的命名空间里通过 [Helm](https://helm.sh) 库来创建一个 Helm `release` 对象。`helm` CLI 也能够被用来检查安装的 Kubernetes 清单以获取更多细节。参阅 Helm 文档来了解如何[安装 Helm](https://helm.sh/docs/intro/install/)。

```console
$ helm get manifest osm --namespace <osm-namespace>
```

## 组件

让我们看一下各个组件：

### 1 代理控制平面

代理控制平面（Proxy Control Plane）组件在操作[服务网格](https://www.bing.com/search?q=What%27s+a+service+mesh%3F)上扮演了重要的角色。所有的代理被作为 [sidecar](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) 来安装，并且和代理控制平面建立了一个 HTTP 连接。这些代理持续接收配置更新。这个组件实现了被指定的反向代理所需要的接口。osm-edge 实现了 [Pipy Repo](https://flomesh.io/docs/en/operating/repo/0-intro)。

### 2 证书管理器

证书管理器是一个组件，它负责给每个参与到服务网格的服务提供一个 TLS 证书。这些服务证书被用来通过 mTLS 来建立和加密服务之间的连接。

### 3 端点提供程序

端点提供程序是一个或者多个组件，用来和参与到服务网格的计算平台（Kubernetes 集群、本地机器，或者云提供商的虚拟机）进行通信。端点提供程序解析服务的名字为 IP 地址列表。端点提供程序理解计算提供商所为之实现的特定的基本元素，例如虚拟机、虚拟机扩展集和 Kubernetes 集群。

### 4 网格规范

网格规范是一个围绕在现有 [SMI Spec](https://github.com/deislabs/smi-spec) 组件的封装。这个组件抽象了为 YAML 定义所选择的特定存储。这个模块是围绕 [SMI Spec's Kubernetes informers](https://github.com/deislabs/smi-sdk-go) 的有效封装，当前抽象了其存储 (Kubernetes/etcd) 规范。

### 5 网格目录

网格目录是 osm-edge 组件的中心，其把全部其他组件的输出组合到一个结构中，该结构然后被转换为代理配置，通过代理控制平面派发到所有正在监听的代理中。

这个组件：

1. 同[网格规范模块（4）](#4-网格规范)通信，通过 [SMI Spec](https://github.com/deislabs/smi-spec) 来检测一个 [Kubernetes 服务](https://kubernetes.io/docs/concepts/services-networking/service/)的创建，改变或者删除。
2. 触及[证书管理器（2）](#2-证书管理器)并为新发现的服务请求一个新的 TLS 证书。
3. 通过观察计算平台，经由[端点提供程序（3）](#3-端点提供程序)，获取网格工作负载的 IP 地址。
4. 组合如上 1、2 和 3 的输出到一个数据结构，该结构被传送到[代理控制平面（1）](#1-代理控制平面)，序列化并被发送到所有相关的连接代理。

![diagram](https://user-images.githubusercontent.com/2224492/176060685-8504c433-c91b-4f9e-9754-f9ccb6c28a87.png)

### 6 Sidecar 驱动

Sidecar 驱动是 osm-edge 在原 osm 设计的基础上提出的概念，用于实现代理控制平面（Proxy Control Plane）与特定代理解耦，提供更灵活的 Sidecar 接入能力。而代理控制平面仅负责 Sidecar 驱动的生命周期的管理以及接口的适配，不再与 Sidecar 直接交互；Sidecar 厂商基于标准的 Sidecar 驱动接口，按照自有的规范来实现 Sidecar 驱动，为 Sidecar 提供持续的配置和更新的能力。

![Sidecar 驱动生命周期](https://user-images.githubusercontent.com/95846930/175821540-9b7326ac-41e4-4f8e-b23d-bc6b0a5bb7c8.png)


## 详细的组件描述

这个章节勾勒了被采纳的约定和 osm-edge 的开发指导。在这个章节被讨论的组件有：

- (A) 代理 [sidecar](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) —— Pipy、Envoy 或者其他兼容服务网格的反向代理
- (B) [代理证书](#b-代理-tls-证书) —— 特有的 X.509 证书通过[证书管理器](#2-证书管理器)发送到指定的代理
- (C) 服务 —— 在 SMI Spec 中被引用的 [Kubernetes 服务资源](https://kubernetes.io/docs/concepts/services-networking/service/)
- (D) [服务证书](#d-服务-tls-证书) —— 发送给服务的 X.509 证书
- (E) 策略 —— 被目标服务的代理所实施的 [SMI Spec](https://smi-spec.io/) 流量策略
- 对于给定服务，服务端点处理流量的例子：
  - (F) Azure VM —— 运行在一个 Azure VM 上的进程，监听在 IP 1.2.3.11，端口 81 上的连接。
  - (G) Kubernetes Pod —— 运行在一个 Kubernetes 集群上的容器，监听在 IP 1.2.3.12，端口 81 上的连接。
  - (H) 企业内计算 —— 运行在用户的私有数据中心的一台机器上的进程，监听在 IP 1.2.3.13，端口 81 上的连接。

[服务（C）](#c-服务)被分配一个证书（D），同时和一个 SMI Spec 策略相关联（E）。
对于[服务（C）](#c-服务)的流量被端点（F，G，H）处理，这些端点的每一个都被添加了一个代理（A）。

![service-relationships-diagram](https://user-images.githubusercontent.com/2224492/176343499-7b48094f-647e-421b-b349-03556fd0f90a.png)

### (C) 服务

上图中的服务是一个 [Kubernetes 服务资源](https://kubernetes.io/docs/concepts/services-networking/service/)，在 SMI Spec 中被引用。一个例子是下面所定义的 `bookstore` 服务，被一个 `TrafficSplit` 策略引用。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bookstore
  labels:
    app: bookstore
spec:
  ports:
    - port: 14001
      targetPort: 14001
      name: web-port
  selector:
    app: bookstore

---
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-traffic-split
spec:
  service: bookstore
  backends:
    - service: bookstore-v1
      weight: 100
```

### (A) 代理

在 osm-edge 里面，`Proxy` 被定义作为一个抽象逻辑组件，它要：

- 支撑一个服务网格进程（容器或者在 Kubernetes 上运行的二进制程序或者一个 VM）
- 维护一个到代理控制平面的连接
- 持续接收来自[代理控制平面](#1-代理控制平面)（Pipy Repo 实现）的配置更新。osm-edge 提供 [Pipy](https://flomesh.io/) 反向代理实现的开箱即用方案。

### (F,G,H) 端点

在 osm-edge 代码库里面，`Endpoint` 被定义作为一个容器或者一个虚拟机的 IP 地址和端口号的元组，其寄宿着一个代理，这个代理支撑一个进程，这个进程是一个服务的成员，并且作为服务网格的参与者。
[服务端点 (F,G,H)](#fgh-端点) 是实际的二进制程序，其服务于针对[服务 (C)](#c-服务)的流量。
一个端点唯一地标识了一个容器，二进制程序或者一个进程。
它有一个 IP 地址、端口号和给服务的附属物。
一个服务可以有 0 个或者多个端点，每一个端点只能有一个 sidecar 代理。既然一个端点必然属于一个单独的服务，那么一个相关联的代理也必然属于一个单独的服务。

### (D) 服务 TLS 证书

代理，支撑端点，它们构成了一个给定的服务，这个服务将共享证书。这个证书被用来和服务网格中其他服务的对等代理所支撑的端点建立 mTLS 连接。服务证书是短期的。每个服务证书的生命周期[差不多在 24 小时](#证书生命周期)，从而消除了对于一个证书撤销机制的需要。osm-edge 对这类证书声明了一个 `ServiceCertificate` 类型。

在开发者文档的[接口](#接口)章节，`ServiceCertificate` 描述了这类证书如何被引用。

### (E) 策略

在上图 (E) 中被引用的策略组件是任何的引用[服务 (C)](#c-服务)的 [SMI Spec 资源](https://github.com/deislabs/smi-spec#service-mesh-interface)。例如，`TrafficSplit`，应用服务 `bookstore` 和 `bookstore-v1`：

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-traffic-split
spec:
  service: bookstore
  backends:
    - service: bookstore-v1
      weight: 100
```

### 证书生命周期

被[证书管理器](#2-证书管理器)发出的服务证书是短期证书，有效性差不多 24 小时。这个短证书过期机制剔除了对显式的撤销机制的需要。一个给定的证书过期时间将被以 24 小时为基础随机地缩短或者延长，这是为了避免底层证书管理系统遭遇 [thundering herd 问题](https://en.wikipedia.org/wiki/Thundering_herd_problem)。

### 代理和端点的关系

- `Proxy` 是反向代理，其尝试连接到代理控制平面。
- `Endpoint` 是由 `Proxy` 支撑的，并且是一个 `Service` 的成员。osm-edge 或许已经通过[端点提供程序](#3-端点提供程序)发现了端点，这些端点属于一个给定的服务，但是 osm-edge 没有看到任何的代理，支撑这些端点，连接到代理 Control Plane。

![service-mesh-participants](https://user-images.githubusercontent.com/2224492/176342258-8f28b01e-8ef9-49e6-947b-544f9b2739fc.png)
