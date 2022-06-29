---
title: "应用容器启动"
description: "应用容器启动"
type: docs
---

# 容器启动

由于 osm-edge 使用 sidecar 代理注入作为服务网格一部分的应用程序 pod，并设置流量重定向规则以通过 sidecar 代理路由进出 pod 的所有流量，因此在某些情况下，现有应用程序容器可能无法按预期启动。

## 当应用程序容器在启动时依赖于网络连接时

一旦 Pipy sidecar 代理和 `osm-init` 初始化容器被 osm-edge 注入到应用 pod，启动时依赖网络连接的应用容器在启动时可能会遇到问题。这是因为 sidecar 的注入，来自应用容器的所有基于 TCP 的网络流量都会被路由 sidecar 代理并受制于服务网格流量策略。这意味着要像没有注入 sidecar 代理容器一样路由应用流量，osm-edge 控制器必须首先在应用 pod 上对 sidecar 代理进行编程以允许此类流量。如果没有配置 Pipy sidecar 代理，来自应用程序容器的所有流量都将被丢弃。

当 osm-edge 配置为启用宽松流量策略模式时，osm-edge 将在 Pipy sidecar 代理上编写通配符流量策略规则，以允许每个 pod 访问网格内的所有服务。当 osm-edge 配置为启用 SMI 流量策略模式时，必须配置显式 SMI 策略以启用网格中的应用程序之间的通信。

不管流量策略模式如何，在启动时依赖于网络连接的应用程序容器如果对网络准备就绪的延迟没有弹性，则在启动时可能会遇到问题。注入 Pipy 代理 sidecar 后，仅当 osm-edge 控制器对 Sidecar 代理进行编程以允许应用程序流量通过网络时，才认为网络已准备就绪。

建议应用程序容器对应用程序 pod 中 Pipy 代理 sidecar 的初始引导阶段具有足够的弹性。

需要注意的是，[容器的重启策略](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)也会影响应用容器的启动。如果应用程序容器的启动策略设置为 `Never`，并且它依赖于网络连接在启动时准备好，则容器可能无法访问网络，直到 Pipy 代理 sidecar 准备好允许应用程序容器访问网络，从而导致应用程序容器退出并且永远无法从失败的启动中恢复。因此，如果应用程序容器在启动时依赖于网络连接，建议不要使用 `Never` 的容器重启策略。

### 相关问题（进行中的工作）

- [Kubernetes issue 65502](https://github.com/kubernetes/kubernetes/issues/65502)：支持同一容器容器之间的启动依赖关系

