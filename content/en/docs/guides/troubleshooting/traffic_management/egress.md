---
title: "Egress Troubleshooting"
description: "Egress Troubleshooting Guide"
---

## When Egress is not working as expected

### 1. Confirm egress is enabled

Confirm egress is enabled by verifying the value for the `enableEgress` key in the `osm-mesh-config` `MeshConfig` custom resource. `osm-mesh-config` resides in the namespace osm-edge control plane namespace (`osm-system` by default).

```console
# Returns true if egress is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enableEgress}{"\n"}'
true
```

The above command must return a boolean string (`true` or `false`) indicating if egress is enabled.

### 2. Inspect osm-edge controller logs for errors

```bash
# When osm-controller is deployed in the osm-system namespace
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

Errors will be logged with the `level` key in the log message set to `error`:
```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. Confirm the Pipy configuration

Check that egress is enabled in the configuration used by the Pod's sidecar.

```json
{
  "Spec": {
    "SidecarLogLevel": "error",
    "Traffic": {
      "EnableEgress": true
    }
  }
}
```
