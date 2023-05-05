---
title: "Egress Gateway"
description: "Mannage access to the Internet and services external to the service mesh with Egress gateway."
weight: 7
---

# Egress Gateway

Egress gateway is another approach to manage access to services external to the service mesh.

In this mode, the sidecar forwards egress traffic to the Egress gateway, and Egress gateway completes the forwarding to external services.

Using an Egress Gateway provides unified egress management, although it is an extra hop from the network perspective. The security team can set network rules on a fixed device to allow access to external services. The node selector is then used when the egress gateway is dispatched to these devices. Both approaches have their advantages and disadvantages and need to be chosen based on specific scenarios.

## Configuration Egress Gateway

Egress gateway also supports the enable and disable mesh-wide passthrough, you can refer to configuration section of [Egress](/docs/guides/traffic_management/egress#configuring-egress).

First of all, it's required to deploy the egress gateway. Refer to [Egress Gateway Demo]() for egress gateway installation.

Once we have the gateway, we need to add a global egress policy. The spec of `EgressGateway` declares that egress traffic can be forwarded to the Service `global-egress-gateway` under the namespace `egress-gateway`.

```yaml
kind: EgressGateway
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: global-egress-gateway
  namespace: fsm
spec:
  global:
    - service: fsm-egress-gateway
      namespace: fsm
```

`global-egress-gateway` created above is a global egress gateway. By default, all egress traffic will be redirected to this global egress gateway by sidecar.

## More configuration for Egress gateway

As we know, the sidecar will forward egress traffic to egress gateway and the latter one will complete the forwarding to services external to mesh.

The transmission between sidecar and egress gateway has two modes: `http2tunnel` and `socks5`. This can be set during the deployment of egress gateway and it will use `http2tunnel` if omitted.

More details about this, please refer to [FSM Egress Gateway Transmission Mode]().

## Demo

To learn more about configuration for egress gateway, refer to following demo guides:

- [Egress gateway passthrough](/docs/demos/egress_gateway_passthrough)
- [Egress gateway policy](/docs/demos/egress_gateway_policy)

```