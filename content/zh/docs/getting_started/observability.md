---
title: "用 Prometheus 和 Grafana 配置可观测性"
description: "使用 osm-edge 带 Prometheus 和 Grafana 的可观测性集成来检查书店应用间的流量"
type: docs
weight: 5
---

# 用 Prometheus 和 Grafana 来配置可观测性

接下来的文章展示了如何安装自带 Prometheus 和 Grafana 栈的 osm-edge，从而具备可观测性和监视能力。对于使用在集群上自有的 Prometheus 和 Grafana 栈协同 osm-edge 的例子，请参阅[集成 osm-edge 到 Prometheus 和 Grafana](/docs/demos/prometheus_grafana/)示例。

在这篇文章中所创建的配置不应该被用于生产环境。对于生产级的部署，请参阅 [Prometheus 运维](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md)和[在 Kubernetes 中部署 Grafana](https://grafana.com/docs/grafana/latest/installation/kubernetes/)。


## 安装带 Prometheus 和 Grafana 的 osm-edge

在 `osm install` 上，一个 Prometheus 和/或 Grafana 实例可以通过默认的 osm-edge 配置来自动提供。
```bash
 osm install --set=osm.deployPrometheus=true \
             --set=osm.deployGrafana=true
```
更多可观测性信息在[可观测性指南](/docs/guides/observability)。

## Prometheus

当配置时带了 `--set=osm.deployPrometheus=true` 标记，osm-edge 安装时将部署一个 Prometheus 实例来抓取 sidecar 和 osm-edge 控制平面指标端点。[抓取配置文件](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/templates/prometheus-configmap.yaml)定义了默认的 Prometheus 行为和被 osm-edge 采集的指标集。

## Grafana

在 `osm install` 上，osm-edge 能够被配置为通过使用 `--set=osm.deployGrafana=true` 标记来部署一个 [Grafana](https://grafana.com/grafana/) 实例。osm-edge 提供预配置的仪表板，这些在可观测性指南的[osm-edge Grafana 仪表板](/docs/guides/observability/metrics/#osm-edge-grafana-仪表板)章节有描述。

## 启动指标抓取

通过使用 `osm metrics` 命令，在命名空间范围内开启指标。默认的，osm-edge **不会**为在网格中的 Pod 配置指标抓取。

```bash
osm metrics enable --namespace test
osm metrics enable --namespace "test1, test2"

```
> 注意：正在启动指标抓取的命名空间必须已经是网格的一部分。

## 检查仪表板

osm-edge Grafana 仪表板能够通过如下命令来查看：

```bash
osm dashboard
```

导航到 [http://localhost:3000](http://localhost:3000) 来访问 Grafana 仪表板。默认的用户名和密码都是 `admin`。在 Grafana 主页上点击 **Home** 图标，将看到一个文件夹，里面包含了 osm-edge 控制平面和数据平面的仪表板。

## 下一步

[清除示例应用并卸载 osm-edge](/docs/getting_started/cleanup/).