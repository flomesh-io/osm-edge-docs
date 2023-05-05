---
title: "数据平面基准测试"
description: "osm-edge 和 osm 数据平面的基准测试"
weight: 1
---

osm-edge 致力于在提供服务网格功能的同时提供高性能资源的服务网格，使得资源受限的边缘环境同样可以使用到云端用到的服务网格功能。

在本测试中，针对 osm-edge（v1.1.0）和 osm（v1.1.0）进行了基准测试。主要关注使用两种不同的网格时的服务 TPS、延迟分布，同时监控数据面的资源开销。

osm-edge 使用 Pipy 作为数据平面；osm 使用 Envoy 作为数据平面。

## 测试环境

该基准测试是在运行在腾讯云 CVM 上的 Kubernetes 集群中测试的。集群共有 2 个标准型S5 的节点，osm-edge 和 osm 均**开启宽松流量模式以及 mTLS，其他为默认设置**。

* Kubernetes：k3s v1.20.14+k3s2
* OS：Ubuntu 20.04
* 节点：16c32g * 2
* 负载生成器：8c16g

测试应用使用常见的 SpringCloud 微服务架构，应用取自 [flomesh-bookinfo-demo](https://github.com/flomesh-io/flomesh-bookinfo-demo/)，这是一个使用 SpringCloud 实现的 bookinfo 应用。在测试中，并没有用到所有的服务，而是选择了其中的 API 网关以及 Bookinfo Ratings 服务。

![](https://user-images.githubusercontent.com/2224492/178288704-3aa44151-4c57-4538-9a0a-55310bb4f200.png)

通过 Ingress 提供服务的对外访问；负载生成器使用了常见的 Apache Jmeter 5.5。

测试中，选择了 2 个链路进行测试：一个是通过 Ingress 直接访问 Bookinfo Ratings 服务（下文简称 ratings ）；另一个中间还会经过 API Gateway（下文简称 gateway-ratings）。之所以选择这 2 个 链路，是为了覆盖单 sidecar 和多 sidear 的场景。

## 过程

在测试开始，我们先进行非网格的测试（即不注入 sidecar，下文简称 non-mesh），然后分别使用 osm-edge 和 osm 网格进行测试。

为了模拟边缘资源受限的场景，使用网格时限制 sidecar 使用的 CPU，分别测试 1 core 和 2 core 的场景。

因此，一共是 5 轮测试，每轮测试 2 个不同的链路。

## 性能

测试时 Jmeter 使用 200 个线程，持续进行 5 分钟的测试。每轮测试前，会先进行 2 分钟的预热。

由于篇幅原因，下面仅展示 gateway-ratings 链路在不同 sidecar 资源限制下的测试结果。

*​注：​表格中的 sidecar 是指 API Gateway 的 sidecar*

| 网格     | Sidecar 最大 CPU | TPS  | 90th | 95th | 99th | sidecar CPU 占用 | sidecar内存占用 |
|----------|:-----------------|------|:-----|:-----|:-----|:-----------------|:----------------|
| non-mesh | NA               | 3283 | 87   | 89   | 97   | NA               | NA              |
| osm-edge | 2                | 3395 | 77   | 79   | 84   | 130%             | 52              |
| osm      | 2                | 2189 | 102  | 104  | 109  | 200%             | 108             |
| osm-edge | 1                | 2839 | 76   | 77   | 79   | 100%             | 34              |
| osm      | 1                | 1097 | 201  | 203  | 285  | 100%             | 105             |


### sidecar 2 core

在 sidecar 限制 2 core 的场景下，使用 osm-edge 网格后对于不使用网格在 TPS 上会有少量的提升，同样延迟也得到了改善。不管是 API Gateway 还是 Bookinfo Ratings 的 sidecar 仍未跑完 2 core（只用到 65%），此时 Bookinfo Ratings 服务本身的性能已至极限。

而 osm 网格的 TPS 下降接近 30%，API Gateway 的 sidecar CPU 已经跑满，为瓶颈所在。

在内存方面，osm-edge 和 osm sidecar 的内存占用分别为：50 MiB 和 105 MiB。

**TPS**

![](https://user-images.githubusercontent.com/2224492/178294418-d3d63aef-8c54-49e4-a40e-8bacdec26f74.png)

**延迟分布**

![](https://user-images.githubusercontent.com/2224492/178294471-b8e1b3c6-a8fd-47cb-872a-0c40418b0da7.png)

**API 网关 sidecar CPU 占用**

![](https://user-images.githubusercontent.com/2224492/178294732-73aaa9f4-e159-4b8e-ab12-521985313358.png)

**API 网关 sidecar 内存占用**

![](https://user-images.githubusercontent.com/2224492/178294829-9d2f0794-12e7-4cd6-827d-11af8b632db9.png)

**Bookinfo Ratings sidecar CPU 占用**

![](https://user-images.githubusercontent.com/2224492/178295086-6380004f-369d-4f6b-afeb-71b48c0e3053.png)

**Bookinfo Ratings sidecar 内存占用**

![](https://user-images.githubusercontent.com/2224492/178295267-004e7676-04b5-4fef-8e5d-ca196bd7dedc.png)

### sidecar 1 core

在限制 sidecar 只能使用 1 core CPU 的测试中，结果差距尤为明显。此时 API Gateway 的 sidecar 成为性能的瓶颈，osm-edge 和 osm 的 sidecar 均耗尽 CPU。

在 TPS 方面，osm-edge 下降 12%，osm 的 TPS 下降达到了惊人的 65%。

**TPS**

![](https://user-images.githubusercontent.com/2224492/178295573-8be92413-d499-476e-b3e1-a23d0bcbcda3.png)

**延迟分布**

![](https://user-images.githubusercontent.com/2224492/178296728-c7ea9a12-d9d4-4be0-9c8d-32bb91724f36.png)

**API 网关 sidecar CPU 占用**

![](https://user-images.githubusercontent.com/2224492/178300176-0a76080b-3bcb-48f4-a506-4a105ad8c4a8.png)

**API 网关 sidecar 内存占用**

![](https://user-images.githubusercontent.com/2224492/178300241-95917e41-5857-4a80-8234-ff6533310ef5.png)

**Bookinfo Ratings sidecar CPU 占用**

![](https://user-images.githubusercontent.com/2224492/178300596-2f767c75-6872-4aa5-b943-a3eeae84c55e.png)

**Bookinfo Ratings sidecar 内存占用**

![](https://user-images.githubusercontent.com/2224492/178300658-53bdf00d-6f2f-484a-8c3f-0399f9b683ed.png)

## 总结

本次主要在限制 sidecar 资源的情况下对 osm-edge 和 osm 的数据平面进行了基准测试，从结果来看 osm-edge 在资源占用低的情况下仍然可以保持较高的性能，对资源的利用更加高效。对于资源受限的边缘场景来说，在较低的资源开销下可以享受云端才能使用的服务网格功能。这些都得益于 [Pipy](https://flomesh.io) 低资源高性能的特点。

当然，osm-edge 适合边缘计算场景，但其同样可以应用于云端。尤其是大规模服务的云端环境，满足对成本控制的要求。