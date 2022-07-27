---
title: "Setup osm-edge"
description: "Install the osm-edge control plane using the osm-edge CLI"
type: docs
weight: 1
---

# Setup osm-edge

## Prerequisites
This demo of osm-edge {{< param osm_edge_version >}} requires:
  - a cluster running Kubernetes {{< param min_k8s_version >}} or greater (using a cloud provider of choice, [minikube](https://minikube.sigs.k8s.io/docs/start/), or similar)
  - a workstation capable of executing [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) scripts
  - [The Kubernetes command-line tool](https://kubernetes.io/docs/tasks/tools/#kubectl) - `kubectl`
  - the [osm-edge code repo](https://github.com/flomesh-io/osm-edge/) available locally

> Note: This document assumes you have already installed credentials for a Kubernetes cluster in ~/.kube/config and `kubectl cluster-info` executes successfully.



## Download and install the osm-edge command-line tool

The `osm` command-line tool contains everything needed to install and configure Open Service Mesh.
The binary is available on the [osm-edge GitHub releases page](https://github.com/flomesh-io/osm-edge/releases/).

### GNU/Linux

Download the 64-bit GNU/Linux or macOS binary of osm-edge {{< param osm_edge_version >}}:
```bash
system=$(uname -s)
release={{< param osm_edge_version >}}
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-${release}-${system}-amd64.tar.gz | tar -vxzf -
./${system}-amd64/osm version
```

### macOS

Download the 64-bit macOS binaries for osm-edge {{< param osm_edge_version >}}

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(uname -m)
release={{< param osm_edge_version >}}
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
. /${system}-${arch}/osm version
```

The `osm` CLI can be compiled from source according to [this guide](/docs/guides/cli).

## Installing osm-edge on Kubernetes

With the `osm` binary downloaded and unzipped, we are ready to install Open Service Mesh on a Kubernetes cluster:

The command below shows how to install osm-edge on your Kubernetes cluster.
This command enables
[Prometheus](https://github.com/prometheus/prometheus),
[Grafana](https://github.com/grafana/grafana), and
[Jaeger](https://github.com/jaegertracing/jaeger) integrations.
The `osm.enablePermissiveTrafficPolicy` chart parameter in the `values.yaml` file instructs osm-edge to ignore any policies and
let traffic flow freely between the pods. With Permissive Traffic Policy mode enabled, new pods
will be injected with Envoy, but traffic will flow through the proxy and will not be blocked by access control policies.

> Note: Permissive Traffic Policy mode is an important feature for brownfield deployments, where it may take some time to craft SMI policies. While operators design the SMI policies, existing services will continue to operate as they have been before osm-edge was installed.

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge will be installed
export osm_mesh_name=osm # Replace osm with the desired osm-edge mesh name

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.deployPrometheus=true \
    --set=osm.deployGrafana=true \
    --set=osm.deployJaeger=true
```

Read more on osm-edge's integrations with Prometheus, Grafana, and Jaeger in the [observability documentation](/docs/guides/observability/).

## Next Steps

Now that the osm-edge control plane is up and running, [add applications](/docs/getting_started/install_apps/) to the mesh.
