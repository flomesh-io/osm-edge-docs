---
title: "osm-edge Resource Requests and Limits"
description: "Resource requests and limits for osm-edge pods and deployments"
type: docs
weight: 3
---

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| osm.injector.resource | object | `{"limits":{"cpu":"0.5","memory":"64M"},"requests":{"cpu":"0.3","memory":"64M"}}` | Sidecar injector's container resource parameters |
| osm.osmBootstrap.resource | object | `{"limits":{"cpu":"0.5","memory":"128M"},"requests":{"cpu":"0.3","memory":"128M"}}` | osm-edge bootstrap's container resource parameters |
| osm.osmController.resource | object | `{"limits":{"cpu":"1.5","memory":"1G"},"requests":{"cpu":"0.5","memory":"128M"}}` | osm-edge controller's container resource parameters. See https://docs.openservicemesh.io/docs/guides/ha_scale/scale/ for more details. |
| osm.prometheus.resources | object | `{"limits":{"cpu":"1","memory":"2G"},"requests":{"cpu":"0.5","memory":"512M"}}` | Prometheus's container resource parameters |

> Note: These are the default values and can be configured in [values.yaml](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/values.yaml)