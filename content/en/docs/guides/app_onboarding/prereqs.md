---
title: "Prerequisites"
description: "Application Requirements"
aliases: ["/docs/application_requirements"]
weight: 1
---

# Prerequisites

## Security Contexts

* Do not run applications with the user ID (UID) value of **1500**. This is **reserved** for the Pipy proxy sidecar container injected into pods by osm-edge's sidecar injector.
* If security context `runAsNonRoot` is set to `true` at the pod level, a `runAsUser` value must be provided either for the pod
or for each container. For example:
  ```yaml
    securityContext:
      runAsNonRoot: true
      runAsUser: 1200
  ```
    If the UID is omitted, application containers may attempt to run as root user by default, causing conflict with the pod's security context.
* Additional capabilities are not required.

> Note: the osm-edge init container is programmed to run as root and add capability `NET_ADMIN` as it requires these security
> contexts to finish scheduling. These values are not changed by application security contexts.

## Ports

Do not use the following ports as they are used by the Pipy sidecar.

| Port  | Description                            |
| ----- | -------------------------------------- |
| 15000 | Pipy Admin Port                       |
| 15001 | Pipy Outbound Listener Port           |
| 15003 | Pipy Inbound Listener Port            |
| 15010 | Pipy Prometheus Inbound Listener Port |
