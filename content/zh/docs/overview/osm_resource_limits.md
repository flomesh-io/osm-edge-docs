---
title: "osm-edge 资源需求和限制"
description: "给 osm-edge Pod 和部署的资源需求与限制"
type: docs
weight: 3
---

| 键名 | 类型 | 默认值 | 描述 |
|-----|------|---------|-------------|
| osm.injector.resource | object | `{"limits":{"cpu":"0.5","memory":"64M"},"requests":{"cpu":"0.3","memory":"64M"}}` | Sidecar 注入器的容器资源参数。|
| osm.osmBootstrap.resource | object | `{"limits":{"cpu":"0.5","memory":"128M"},"requests":{"cpu":"0.3","memory":"128M"}}` | osm-edge 引导程序的容器资源参数。|
| osm.osmController.resource | object | `{"limits":{"cpu":"1.5","memory":"1G"},"requests":{"cpu":"0.5","memory":"128M"}}` | osm-edge 控制器的容器资源参数。请参阅 https://osm-edge-docs.flomesh.io/docs/guides/ha_scale/scale/ 以获取更多细节。|
| osm.prometheus.resources | object | `{"limits":{"cpu":"1","memory":"2G"},"requests":{"cpu":"0.5","memory":"512M"}}` | Prometheus 的容器资源参数。|

> 注意：这些都是默认值，其可以通过 [values.yaml](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/values.yaml) 来配置。