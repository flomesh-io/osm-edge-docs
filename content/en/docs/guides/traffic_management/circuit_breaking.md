---
title: "Circuit Breaking"
description: "Using Circuit breaking to limit connections and requests"
type: docs
weight: 11
---

# Circuit Breaking

Circuit breaking is a critical component of distributed systems and an important resiliency pattern. Circuit breaking allows applications to fail quickly and apply back pressure downstream as soon as possible, thereby providing the means to limit the impact of failures across the system. This guide describes how circuit breaking can be configured in osm-edge.

## Configuring circuit breaking

osm-edge leverages its [UpstreamTrafficSetting API][1] to configure circuit breaking attributes for traffic directed to an upstream service. We use the term `upstream service` to refer to a service that receives connections and requests from clients and return responses. The specification enables configuring circuit breaking attributes for an upstream service at the connection and request level.

Circuit breaking is applicable at both the TCP and HTTP level, and can be configured using the `connectionSettings` attribute in the `UpstreamTrafficSetting` resource. TCP traffic settings apply to both TCP and HTTP traffic, while HTTP settings only apply to HTTP traffic.

The following circuit breaking configurations are supported:

- `spec.host`: upstream host.  For a Kubernetes service `my-svc` in the namespace `my-namespace`, the `UpstreamTrafficSetting` resource must be created in the namespace `my-namespace`, and `spec.host` must be an FQDN of the form `my-svc.my-namespace.svc.cluster.local`. When specified as a match in an [Egress policy](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.EgressSpec), `spec.host` must correspond to the host specified in the Egress policy and the `UpstreamTrafficSetting` configuration must belong to the same namespace as the `Egress` resource.

- `connectionSettings.http.circuitBreaking`：HTTP level configuration options
  - `statTimeWindow`: specifies statistical time period of circuit breaking, e.g. `1m`.
  - `minRequestAmount`: specifies minimum number of requests (in an active statistic time span) that can trigger circuit breaking.
  - `errorAmountThreshold`: specifies the amount threshold of error request, e.g `100`.
  - `errorRatioThreshold`: specifies the ratio threshold of error request, e.g `50%`.
  - `slowTimeThreshold`: specifies the time threshold of slow request, e.g. `100ms`.
  - `slowAmountThreshold`: specifies the amount threshold of slow request, e.g. `100`.
  - `slowRatioThreshold`: specifies the ratio threshold of slow request, e.g. `10%`.
  - `degradedTimeWindow`: specifies recovery timeout (in seconds) when circuit breaker opens, e.g. `1m`.
  - `degradedStatusCode`: specifies the degraded http status code of circuit breaking, e.g. `503`
  - `degradedResponseContent`: specifies the degraded http response content of circuit breaking, e.g. `Service Unavailable!`.


```yaml
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-circuit-breaking
spec:
  host: fortio.server.svc.cluster.local
  connectionSettings:
    http:
      circuitBreaking:
        statTimeWindow: 1m
        minRequestAmount: 200
        errorAmountThreshold: 100
        errorRatioThreshold: 0.50
        slowTimeThreshold: 100ms
        slowAmountThreshold: 100
        slowRatioThreshold: 0.10
        degradedTimeWindow: 1m
        degradedStatusCode: 503
        degradedResponseContent: 'Service Unavailable!'
```

## Demo

To learn more about configuring circuit breaking, refer to the following demo:
- [Circuit breaking for destinations within the mesh](/docs/demos/circuit_breaking_mesh)

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec