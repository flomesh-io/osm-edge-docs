---
title: "卸载 osm-edge"
description: "卸载 osm-edge 和书店应用"
type: docs
weight: 6
---

# 卸载 osm-edge

新手上路这个章节里面的文章概述了如何安装 osm-edge 和示例应用。如果要卸载全部与之相关的资源，就需要删除这些示例应用和相关的 SMI 资源，并且卸载掉 osm-edge 控制平面和集群范围内的 osm-edge 资源。

## 删除示例应用

要清理掉示例应用和相关的 SMI 资源，需要删除它们的命名空间。例如：

```bash
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

## 卸载 osm-edge 控制平面

要卸载 osm-edge 控制平面，请使用 `osm unistall mesh`。

```bash
osm uninstall mesh
```

## 卸载 osm-edge 集群范围内的资源

要卸载 osm-edge 集群范围内的资源，请使用 `osm uninstall mesh --delete-cluster-wide-resources`。

```bash
osm uninstall mesh --delete-cluster-wide-resources
```

关于卸载 osm-edge 的更多细节，请参阅 [卸载指南](docs/guides/uninstall/)
