---
title: "osm-edge control plane components"
description: "Components"
type: docs
weight: 2
---

## Inspect osm-edge Components

Some osm-edge components will be installed by default in the chosen namespace, which defaults to `osm-system`. Inspect them by using the following `kubectl` command:

```console
# Replace osm-system with the namespace where osm-edge is installed
$ kubectl get pods,svc,secrets,meshconfigs,serviceaccount --namespace osm-system
```

Some cluster-wide (non-namespaced) osm-edge components will also be installed. Inspect them using the following `kubectl` command:

```console
kubectl get clusterrolebinding,clusterrole,mutatingwebhookconfiguration
```

Under the hood, `osm` is using [Helm](https://helm.sh) libraries to create a Helm `release` object in the control plane Namespace. The Helm `release` name is the mesh-name. The `helm` CLI can also be used to inspect Kubernetes manifests installed in more detail. See the Helm docs for how to [install Helm](https://helm.sh/docs/intro/install/).

```console
$ helm get manifest osm --namespace <osm-namespace>
```

## Components

Let's take a look at each component:

### (1) Proxy Control Plane

The Proxy Control Plane plays a key part in operating the [service mesh](https://www.bing.com/search?q=What%27s+a+service+mesh%3F). All proxies are installed as [sidecars](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) and establish an mTLS gRPC connection to the Proxy Control Plane. The proxies continuously receive configuration updates. This component implements the interfaces required by the specific reverse proxy chosen. osm-edge implements [Pipy Repo](https://flomesh.io/docs/en/operating/repo/0-intro).

### (2) Certificate Manager

Certificate Manager is a component that provides each service participating in the service mesh with a TLS certificate.
These service certificates are used to establish and encrypt connections between services using mTLS.

### (3) Endpoints Providers

Endpoints Providers are one or more components that communicate with the compute platforms (Kubernetes clusters, on-prem machines, or cloud-providers' VMs) participating in the service mesh. Endpoints providers resolve service names into lists of IP addresses. The Endpoints Providers understand the specific primitives of the compute provider they are implemented for, such as virtual machines, virtual machine scale sets, and Kubernetes clusters.

### (4) Mesh specification

Mesh Specification is a wrapper around the existing [SMI Spec](https://github.com/deislabs/smi-spec) components. This component abstracts the specific storage chosen for the YAML definitions. This module is effectively a wrapper around [SMI Spec's Kubernetes informers](https://github.com/deislabs/smi-sdk-go), currently abstracting away the storage (Kubernetes/etcd) specifics.

### (5) Mesh catalog

Mesh Catalog is the central component of osm-edge, which combines the outputs of all other components into a structure, which can then be transformed into proxy configuration and dispatched to all listening proxies via the proxy control plane.
This component:

1. Communicates with the [mesh specification module (4)](#4-mesh-specification) to detect when a [Kubernetes service](https://kubernetes.io/docs/concepts/services-networking/service/) was created, changed, or deleted via [SMI Spec](https://github.com/deislabs/smi-spec).
2. Reaches out to the [certificate manager (2)](#2-certificate-manager) and requests a new TLS certificate for the newly discovered service.
3. Retrieves the IP addresses of the mesh workloads by observing the compute platforms via the [endpoints providers (3)](#3-endpoints-providers).
4. Combines the outputs of 1, 2, and 3 above into a data structure, which is then passed to the [proxy control plane (1)](#1-proxy-control-plane), serialized and sent to all relevant connected proxies.

![diagram](https://user-images.githubusercontent.com/2224492/176060685-8504c433-c91b-4f9e-9754-f9ccb6c28a87.png)

### 6 Sidecar Driver

The Proxy Control plane component will be responsible for the life cycle management of sidecar drivers and communicating with drivers using the **Sidecar Driver** interface.

Vendors wishing to extend or provide a sidecar will implement the abstraction and provide configuration updates to their respective sidecar implementations in format/manners which suit their sidecar-specific needs.

A Sidecar Driver needs to implement the **Driver** interface to integrate with the OSM _Injector_ and _Controller_.

![Sidecar Driver lifecycle](https://user-images.githubusercontent.com/95846930/175821540-9b7326ac-41e4-4f8e-b23d-bc6b0a5bb7c8.png)

## Detailed component description

This section outlines the conventions adopted and guiding the development of the Open Service Mesh (osm-edge). Components discussed in this section:

- (A) Proxy [sidecar](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) - Pipy or other reverse-proxy with service-mesh capabilities
- (B) [Proxy Certificate](#b-proxy-tls-certificate) - unique X.509 certificate issued to the specific proxy by the [Certificate Manager](#2-certificate-manager)
- (C) Service - [Kubernetes service resource](https://kubernetes.io/docs/concepts/services-networking/service/) referenced in SMI Spec
- (D) [Service Certificate](#d-service-tls-certificate) - X.509 certificate issued to the service
- (E) Policy - [SMI Spec](https://smi-spec.io/) traffic policy enforced by the target service's proxy
- Examples of service endpoints handling traffic for the given service:
  - (F) Azure VM - process running on an Azure VM, listening for connections on IP 1.2.3.11, port 81.
  - (G) Kubernetes Pod - container running on a Kubernetes cluster, listening for connections on IP 1.2.3.12, port 81.
  - (H) On-prem compute - process running on a machine within the customer's private data center, listening for connections on IP 1.2.3.13, port 81.

The [Service (C)](#c-service) is assigned a Certificate (D) and is associated with an SMI Spec Policy (E).
Traffic for [Service (C)](#c-service) is handled by the Endpoints (F, G, H) where each endpoint is augmented with a Proxy (A).
The Proxy (A) has a dedicated Certificate (B), which is different from the Service Cert (D), and is used for mTLS connection from the Proxy to the [proxy control plane](#1-proxy-control-plane).

![service-relationships-diagram](https://user-images.githubusercontent.com/2224492/176343499-7b48094f-647e-421b-b349-03556fd0f90a.png)

### (C) Service

Service in the diagram above is a [Kubernetes service resource](https://kubernetes.io/docs/concepts/services-networking/service/) referenced in SMI Spec. An example is the `bookstore` service defined below and referenced by a `TrafficSplit` policy:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bookstore
  labels:
    app: bookstore
spec:
  ports:
    - port: 14001
      targetPort: 14001
      name: web-port
  selector:
    app: bookstore

---
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-traffic-split
spec:
  service: bookstore
  backends:
    - service: bookstore-v1
      weight: 100
```

### (A) Proxy

In osm-edge `Proxy` is defined as an abstract logical component, which:

- fronts a mesh service process (container or binary running on Kubernetes or a VM)
- maintains a connection to a proxy control plane (xDS server)
- continuously receives configuration updates (xDS protocol buffers) from the [proxy control plane](#1-proxy-control-plane) (Pipy Repo implementation)
  osm-edge ships out of the box with [Pipy](https://flomesh.io/) reverse-proxy implementation.

### (F,G,H) Endpoint

Within the osm-edge codebase `Endpoint` is defined as the IP address and port number tuple of a container or a virtual machine, which is hosting a proxy, which is fronting a process, which is a member of a service and as such participates in the service mesh.
The [service endpoints (F,G,H)](#fgh-endpoint) are the actual binaries serving traffic for the [service (C)](#c-service).
An endpoint uniquely identifies a container, binary, or a process.
It has an IP address, port number, and belongs to a service.
A service can have zero or more endpoints, and each endpoint can have only one sidecar proxy. Since an endpoint must belong to a single service, it follows that an associated proxy must also belong to a single service.

### (D) Service TLS certificate

Proxies, fronting endpoints, which form a given service will share the certificate for the given service.
This certificate is used to establish mTLS connection with peer proxies fronting endpoints of **other services** within the service mesh.
The service certificate is short-lived.
Each service certificate's lifetime will be [approximately 24 hours](#certificate-lifetime), which eliminates the need for a certificate revocation facility.
osm-edge declares a type `ServiceCertificate` for these certificates.
`ServiceCertificate` is how this kind of certificate is referred to in the [Interfaces](#interfaces) section of the developer documentation.

### (E) Policy

The policy component referenced in the diagram above (E) is any [SMI Spec resource](https://github.com/deislabs/smi-spec#service-mesh-interface) referencing the [service (C)](#c-service). For instance, `TrafficSplit`, referencing a services `bookstore`, and `bookstore-v1`:

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-traffic-split
spec:
  service: bookstore
  backends:
    - service: bookstore-v1
      weight: 100
```

### Certificate lifetime

The service certificates issued by the [Certificate Manager](#2-certificate-manager) are short-lived certificates, with a validity of approximately 24 hours.
The short certificate expiration eliminates the need for an explicit revocation mechanism.
A given certificate's expiration will be randomly shortened or extended from the 24 hours, in order to avoid [thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem) inflicted on the underlying certificate management system. Proxy certificates, on the other hand, are long-lived certificates.

### Proxy Certificate, Proxy, and Endpoint relationship

- `ProxyCertificate` is issued by osm-edge for a `Proxy`, which is expected to connect to the proxy control plane sometime in the future. After the certificate is issued, and before the proxy connects to the proxy control plane, the certificate is in the `unclaimed` state. The state of the certificate changes to `claimed` after a proxy has connected to the control plane using the certificate.
- `Proxy` is the reverse-proxy, which attempts to connect to the proxy control plane; the `Proxy` may, or may not be allowed to connect to the proxy control plane.
- `Endpoint` is fronted by a `Proxy`, and is a member of a `Service`. osm-edge may have discovered endpoints, via the [endpoints providers](#3-endpoints-providers), which belong to a given service, but osm-edge has not seen any proxies, fronting these endpoints, connect to the proxy control plane yet.

The **intersection** of the set of issued `ProxyCertificates` ∩ connected `Proxies` ∩ discovered `Endpoints` is the set of participants in the service mesh.

![service-mesh-participants](https://user-images.githubusercontent.com/2224492/176342258-8f28b01e-8ef9-49e6-947b-544f9b2739fc.png)
