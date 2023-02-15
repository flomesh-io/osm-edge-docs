---
title: "Ingress"
description: "Using Ingress to manage external access to services within the cluster"
type: docs
weight: 5
---

# Ingress

## Using Ingress to manage external access to services within the cluster

Ingress refers to managing external access to services within the cluster, typically HTTP/HTTPS services. osm-edge's ingress capability allows cluster administrators and application owners to route traffic from clients external to the service mesh to service mesh backends using a set of rules depending on the mechanism used to perform ingress.

## IngressBackend API

osm-edge leverages its [IngressBackend API][1] to configure a backend service to accept ingress traffic from trusted sources. The specification enables configuring how specific backends must authorize ingress traffic depending on the protocol used, HTTP or HTTPS. When the backend protocol is `http`, the specified source kind must either be: 1. `Service` kind whose endpoints will be authorized to connect to the backend, or 2. `IPRange` kind that specifies the source IP CIDR range authorized to connect to the backend. When the backend protocol is `https`, the source specified must be an `AuthenticatedPrincipal` kind which defines the Subject Alternative Name (SAN) encoded in the client's certificate that the backend will authenticate. A source with the kind `Service` or `IPRange` is optional for `https` backends, and if specified implies that the client must match the source in addition to its `AuthenticatedPrincipal` value. For `https` backends, client certificate validation is performed by default and can be disabled by setting `skipClientCertValidation: true` in the `tls` field for the backend. The `port.number` field for a `backend` service in the `IngressBackend` configuration must correspond to the `targetPort` of a Kubernetes service.

Note that when the `Kind` for a source in an `IngressBackend` configuration is set to `Service`, osm-edge controller will attempt to discover the endpoints of that service. For osm-edge to be able to discover the endpoints of a service, the namespace in which the service resides needs to be a monitored namespace. Enable the namespace to be monitored using:

```bash
kubectl label ns <namespace> openservicemesh.io/monitored-by=<mesh name>
```

### Examples

The following IngressBackend configuration will allow access to the `foo` service on port `80` in the `test` namespace only if the source originating the traffic is an endpoint of the `myapp` service in the `default` namespace:
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http
  sources:
    - kind: Service
      namespace: default
      name: myapp
```

The following IngressBackend configuration will allow access to the `foo` service on port `80` in the `test` namespace only if the source originating the traffic has an IP address that belongs to the CIDR range `10.0.0.0/8`:
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http
  sources:
    - kind: IPRange
      name: 10.0.0.0/8
```

The following IngressBackend configuration will allow access to the `foo` service on port `80` in the `test` namespace only if the source originating the traffic encrypts the traffic with `TLS` and has the Subject Alternative Name (SAN) `client.default.svc.cluster.local` encoded in its client certificate:
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80
        protocol: https # https implies TLS
      tls:
        skipClientCertValidation: false # mTLS (optional, default: false)
  sources:
    - kind: AuthenticatedPrincipal
      name: client.default.svc.cluster.local
```

Refer to the following sections to understand how the `IngressBackend` configuration looks like for `http` and `https` backends.

## Choices to perform Ingress

osm-edge supports multiple options to expose mesh services externally using ingress which are described in the following sections. osm-edge has been tested with Contour and OSS Nginx, which work with the ingress controller installed outside the mesh and provisioned with a certificate to participate in the mesh.

> Note: osm-edge integration with Nginx Plus has not been fully tested for picking up a self-signed mTLS certificate from a Kubernetes secret. However, an alternative way to incorporate Nginx Plus or any ingress is to install it in the mesh so that it is injected with an Pipy sidecar, which will allow it to participate in the mesh. Additional inbound ports such as 80 and 443 may need to be allowed to bypass the Pipy sidecar.

### 1. Using FSM ingress controllers and gateways

Using [FSM](https://github.com/flomesh-io/fsm) ingress controllers and edge proxy is the preferred method for executing Ingress in an osm-edge managed services mesh. Using FSM, users get a high-performance ingress controller with rich policy specifications for a variety of scenarios, while maintaining lightweight profiles.

To use FSM as an ingress, enable it during mesh installation at

```bash
osm install --set fsm.enabled=true
```

In addition to configuring the edge proxy for FSM using the appropriate API, the service mesh backend in osm-edge will only accept traffic from authorized edge proxy or gateways. osm-edge's [IngressBackend specification][1] allows cluster administrators and application owners to explicitly specify how the service mesh backend should authorize ingress traffic. The following sections describe how to use the `IngressBackend` and `HTTPProxy` APIs in combination to allow HTTP and HTTPS ingress traffic to be routed to the mesh backend.

It is recommended that ingress traffic always be restricted to authorized clients. To do this, enable osm-edge to monitor the endpoints of the FSM edge proxy located in the namespace where the osm-edge installation is located: ``bash

```bash
kubectl label ns <osm namespace> openservicemesh.io/monitored-by=<mesh name>
```

#### HTTP Ingress using FSM

A minimal [HTTPProxy][2] configuration and osm-edge's `IngressBackend`[1] specification to route ingress traffic to the mesh service `foo` in the namespace `test` might look like the following:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fsm-ingress
  namespace: test
spec:
  ingressClassName: pipy
  rules:
  - host: foo-basic.bar.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: foo
            port:
              number: 80
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http # http implies no TLS
  sources:
    - kind: Service
      namespace: osm-system
      name: fsm-ingress-pipy-controller
```

The above configuration allows external clients to access the `foo` service under the `test` namespace.

1. The Ingress configuration will route incoming HTTP traffic from external sources with the `Host:` header of `foo-basic.bar.com` to the service named `foo` on port `80` in the `test` namespace.
2. IngressBackend is configured to allow only endpoints named `fsm-ingress-pipy-controller` service from the same namespace where osm-edge is installed (default is `osm-system`) to access port `80` of the `foo` serivce under the `test` namespace.

#### Examples

Refer to the [Ingress with FSM demo](/docs/demos/ingress_fsm) for examples on how to expose mesh services externally using FSM in osm-edge.

### 2. Bring your own Ingress Controller and Gateway

If using osm-edge with FSM for ingress is not feasible for your use case, osm-edge provides the facility to use your own ingress controller and edge gateway for routing external traffic to service mesh backends. Much like how ingress is configured above, in addition to configuring the ingress controller to route traffic to service mesh backends, an IngressBackend configuration is required to authorize clients responsible for proxying traffic originating externally.

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.IngressBackendSpec