---
title: "Traffic Access Control"
description: "Traffic access control using SMI Traffic Access Control API"
weight: 10
---

# Traffic Access Control

The [SMI Traffic Access Control API](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md) can be used to configure access to specific pods and routes based on the identity of a client for locking down applications to only allowed users and services. This allow users to define access control policy for their application based on service identity using Kubernetes service accounts.

> Traffic Access Control API handles the authorization side only.

## What is supported

osm-edge implements the [SMI Traffic Access Control v1alpha3 version](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md).

It supports the following:

- SMI access control policies to authorize traffic access between service identities
- SMI traffic specs policies to define routing rules to associate with access control policies

## How it works

A `TrafficTarget` associates a set of traffic definitions (rules) with a service identity which is allocated to a group of pods.  Access is controlled
via referenced `TrafficSpecs` and by a list of source service identities.  If a pod which holds the reference service identity makes a call to the destination on one of the defined routes then access will be allowed. Any pod which attempts to connect and is not in
the defined list of sources will be denied.  Any pod which is in the defined list but attempts to connect on a route which is not in the list of `TrafficSpecs` will be denied.

```yaml
kind: TCPRoute
metadata:
  name: the-routes
spec:
  matches:
    ports:
    - 8080
---
kind: HTTPRouteGroup
metadata:
  name: the-routes
spec:
  matches:
  - name: metrics
    pathRegex: "/metrics"
    methods:
    - GET
  - name: everything
    pathRegex: ".*"
    methods: ["*"]
```

For this definition, there are two routes: `metrics` and `everything`. It is a common use case to restrict access to `/metrics` to only be scraped by Prometheus. To define the target for this traffic, it takes a `TrafficTarget`.

```yaml
---
kind: TrafficTarget
metadata:
  name: path-specific
  namespace: default
spec:
  destination:
    kind: ServiceAccount
    name: service-a
    namespace: default
  rules:
  - kind: TCPRoute
    name: the-routes
  - kind: HTTPRouteGroup
    name: the-routes
    matches:
    - metrics
  sources:
  - kind: ServiceAccount
    name: prometheus
    namespace: default
```

This example selects all the pods which have the `service-a` `ServiceAccount`. Traffic destined on a path `/metrics` is allowed. The `matches` field is
optional and if omitted, a rule is valid for all the matches in a traffic spec (a OR relationship).  It is possible for a service to expose multiple ports,
the TCPRoute/UDPRoute `matches.ports` field allows the user to specify specifically which port traffic should be allowed on. The `matches.ports` is an optional element, if not specified, traffic will be allowed to all ports on the destination service.

Allowing destination traffic should only be possible with permission of the service owner. Therefore, RBAC rules should be configured to control the pods
which are allowed to assign the `ServiceAccount` defined in the TrafficTarget destination.

**Note:** access control is *always* enforced on the *server* side of a connection (or the target). It is up to implementations to decide whether they would also like to enforce access control on the *client* (or source) side of the connection as well.

Source identities which are allowed to connect to the destination is defined in the sources list.  Only pods which have a `ServiceAccount` which is named in the sources list are allowed to connect to the destination.

## Example implementation for L7

The following implementation shows four services `api`, `website`, `payment` and `prometheus`. It shows how it is possible to write fine grained `TrafficTargets` which allow access to be controlled by route and source.

```yaml
kind: TCPRoute
metadata:
  name: api-service-port
spec:
  matches:
    ports:
    - 8080
---
kind: HTTPRouteGroup
metadata:
  name: api-service-routes
spec:
  matches:
  - name: api
    pathRegex: /api
    methods: ["*"]
  - name: metrics
    pathRegex: /metrics
    methods: ["GET"]
---
kind: TrafficTarget
metadata:
  name: api-service-metrics
  namespace: default
spec:
  destination:
    kind: ServiceAccount
    name: api-service
    namespace: default
  rules:
  - kind: TCPRoute
    name: api-service-port
  - kind: HTTPRouteGroup
    name: api-service-routes
    matches:
    - metrics
  sources:
  - kind: ServiceAccount
    name: prometheus
    namespace: default
---
kind: TrafficTarget
metadata:
  name: api-service-api
  namespace: default
spec:
  destination:
    kind: ServiceAccount
    name: api-service
    namespace: default
  rules:
  - kind: TCPRoute
    name: api-service-port
  - kind: HTTPRouteGroup
    name: api-service-routes
    matches:
    - api
  sources:
  - kind: ServiceAccount
    name: website-service
    namespace: default
  - kind: ServiceAccount
    name: payments-service
    namespace: default
```

The previous example would allow the following HTTP traffic:

| source            | destination   | path     | method |
| ----------------- | ------------- | -------- | ------ |
| website-service   | api-service   | /api     | *      |
| payments-service  | api-service   | /api     | *      |
| prometheus        | api-service   | /metrics | GET    |

## Example implementation for L4

The following implementation shows how to define TrafficTargets for allowing TCP and UDP traffic to specific ports.

```yaml
kind: TCPRoute
metadata:
  name: tcp-ports
spec:
  matches:
    ports:
    - 8301
    - 8302
    - 8300
---
kind: UDPRoute
metadata:
  name: udp-ports
spec:
  matches:
    ports:
    - 8301
    - 8302
---
kind: TrafficTarget
metadata:
  name: protocal-specific
spec:
  destination:
    kind: ServiceAccount
    name: server
    namespace: default
  rules:
  - kind: TCPRoute
    name: tcp-ports
  - kind: UDPRoute
    name: udp-ports
  sources:
  - kind: ServiceAccount
    name: client
    namespace: default
```

> Above configuration will allow TCP and UDP traffic to both `8301` and `8302` ports, but will block UDP traffic to `8300`.

Refer to a guide on [configure traffic policies](/docs/getting_started/traffic_policies/) to learn more.