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

## Sidecar resource requests and limits

One can impose limits on the resources used by the osm-edge sidecar containers via below three ways:

- globally to all mesh managed containers via `MeshConfig`
- containers running under mesh managed specific Namespace via `Namespace limits specs`
- containers running under a specific pod via `Pod limits specs`

### Priority

Pod Annotation > Namespace Annotation > MeshConfig

### Annotations

- openservicemesh.io/sidecar-resource-limits-cpu
- openservicemesh.io/sidecar-resource-limits-memory
- openservicemesh.io/sidecar-resource-limits-storage
- openservicemesh.io/sidecar-resource-limits-ephemeral-storage
- openservicemesh.io/sidecar-resource-requests-cpu
- openservicemesh.io/sidecar-resource-requests-memory
- openservicemesh.io/sidecar-resource-requests-storage
- openservicemesh.io/sidecar-resource-requests-ephemeral-storage


#### Via Meshconfig

```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"sidecar":{"resources":{"limits":{"memory":"2048M"},"requests":{"memory":"1024M"}}}}}' --type=merge
```

#### Via Namespace annotations

```bash
kubectl patch namespace "some_namespace" -p '{"metadata":{"annotations":{"openservicemesh.io/sidecar-resource-limits-memory":"1024M","openservicemesh.io/sidecar-resource-requests-memory":"512M"}}}' --type=merge
```

#### Via Pod annotations

```bash
kubectl patch deployment -n "some_namespace" "deployment_name" -p '{"spec": {"template": {"metadata": {"annotations": {"openservicemesh.io/sidecar-resource-limits-memory": "512M","openservicemesh.io/sidecar-resource-requests-memory": "512M"}}}}}' --type=merge
```

> After changes, pods need to be restarted for changes to take effect.