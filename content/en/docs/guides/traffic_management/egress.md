---
title: "Egress"
description: "Enable access to the Internet and services external to the service mesh."
type: docs
weight: 6
---

# Egress

## Allowing access to the Internet and out-of-mesh services (Egress)

This document describes the steps required to enable access to the Internet and services external to the service mesh, referred to as `Egress` traffic.

osm-edge redirects all outbound traffic from a pod within the mesh to the pod's sidecar proxy. Outbound traffic can be classified into two categories:

1. Traffic to services within the mesh cluster, referred to as in-mesh traffic
2. Traffic to services external to the mesh cluster, referred to as egress traffic

While in-mesh traffic is routed based on L7 traffic policies, egress traffic is routed differently and is not subject to in-mesh traffic policies. osm-edge supports access to external services as a passthrough without subjecting such traffic to filtering policies.

## Configuring Egress

There are two mechanisms to configure Egress:

1. Using the Egress policy API: to provide fine grained access control over external traffic
2. Using the mesh-wide global egress passthrough setting: the setting is toggled on or off and affects all pods in the mesh, enabling which allows traffic destined to destinations outside the mesh to egress the pod.

## 1. Configuring Egress policies

osm-edge supports configuring fine grained policies for traffic destined to external endpoints using its [Egress policy API][1]. To use this feature, enable it if not enabled:

```bash
# Replace osm-system with the namespace where osm-edge is installed
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

Refer to the [Egress policy demo](/docs/demos/egress_policy) and [API documentation][1] on how to configure policies for routing egress traffic for various protocols.

## 2. Configuring mesh-wide Egress passthrough

### Enabling mesh-wide Egress passthrough to external destinations

Egress can be enabled mesh-wide during osm-edge install or post install. When egress is enabled mesh-wide, outbound traffic from pods are allowed to egress the pod as long as the traffic does not match in-mesh traffic policies that otherwise deny the traffic.

1. During osm-edge install (default `osm.enableEgress=false`):

   ```bash
   osm install --set osm.enableEgress=true
   ```

2. After osm-edge has been installed:

   `osm-controller` retrieves the egress configuration from the `osm-mesh-config` `MeshConfig` custom resource in the osm mesh control plane namespace (`osm-system` by default). Use `kubectl patch` to set `enableEgress` to `true` in the `osm-mesh-config` resource.

   ```bash
   # Replace osm-system with the namespace where osm-edge is installed
   kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":true}}}' --type=merge
   ```

### Disabling mesh-wide Egress passthrough to external destinations

Similar to enabling egress, mesh-wide egress can be disabled during osm-edge install or post install.

1. During osm-edge install:

   ```bash
   osm install --set osm.enableEgress=false
   ```

2. After osm-edge has been installed:
   Use `kubectl patch` to set `enableEgress` to `false` in the `osm-mesh-config` resource.
   ```bash
   # Replace osm-system with the namespace where osm-edge is installed
   kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
   ```

With egress disabled, traffic from pods within the mesh will not be able to access external services outside the cluster.

### How it works

When egress is enabled mesh-wide, osm-edge controller programs every Pipy proxy sidecar in the mesh with a wildcard rule that matches outbound destinations that do not correspond to in-mesh services. The wildcard rule that matches such external traffic simply proxies the traffic as is to its original destination without subjecting them to L4 or L7 traffic policies.

osm-edge supports egress for traffic that uses TCP as the underlying transport. This includes raw TCP traffic, HTTP, HTTPS, gRPC etc.

Since mesh-wide egress is a global setting and operates as a passthrough to unknown destinations, fine grained access control (such as applying TCP or HTTP routing policies) over egress traffic is not possible.

Refer to the [Egress passthrough demo](/docs/demos/egress_passthrough) to learn more.

#### Pipy configurations

When egress is enabled globally in the mesh, the osm-edge controller issues the following configuration for each Pipy proxy sidecar.

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

The Pipy script for `EnableEgress=true` will use the original destination logic to route the request to proxy it to the original destination.

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.EgressSpec
