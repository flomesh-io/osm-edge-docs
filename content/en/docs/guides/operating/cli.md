---
title: "Install the osm-edge CLI"
description: "This section describes installing and using the `osm` CLI."
type: docs
weight: 1
---

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater

## Set up the osm-edge CLI

### From the Binary Releases

Download platform specific compressed package from the [Releases page](https://github.com/flomesh-io/osm-edge/releases).
Unpack the `osm` binary and add it to `$PATH` to get started.

#### Linux and macOS

In a bash-based shell on Linux/macOS or [Windows Subsystem for Linux](https://docs.microsoft.com/windows/wsl/about), use `curl` to download the osm-edge release and then extract with `tar` as follows:

```console
# Specify the osm-edge version that will be leveraged throughout these instructions
OSM_VERSION={{< param osm_edge_version >}}

# Linux curl command only
curl -sL "https://github.com/flomesh-io/osm-edge/releases/download/$OSM_VERSION/osm-$OSM_VERSION-linux-amd64.tar.gz" | tar -vxzf -

# macOS curl command only
curl -sL "https://github.com/flomesh-io/osm-edge/releases/download/$OSM_VERSION/osm-$OSM_VERSION-darwin-amd64.tar.gz" | tar -vxzf -
```

The `osm` client binary runs on your client machine and allows you to manage osm-edge in your Kubernetes cluster. Use the following commands to install the osm-edge `osm` client binary in a bash-based shell on Linux or [Windows Subsystem for Linux](https://docs.microsoft.com/windows/wsl/about). These commands copy the `osm` client binary to the standard user program location in your `PATH`.

```console
sudo mv ./linux-amd64/osm /usr/local/bin/osm
```

For **macOS** use the following commands:

```console
sudo mv ./darwin-amd64/osm /usr/local/bin/osm
```

You can verify the `osm` client library has been correctly added to your path and its version number with the following command.

```console
osm version
```
### From Source (Linux, MacOS)

Building osm-edge from source requires more steps but is the best way to test the latest changes and useful in a development environment.

You must have a working [Go](https://golang.org/doc/install) environment.

```console
$ git clone git@github.com:flomesh-io/osm-edge.git
$ cd osm
$ make build-osm
```

`make build-osm` will fetch any required dependencies, compile `osm` and place it in `bin/osm`. Add `bin/osm` to `$PATH` so you can easily use `osm`.

## Install osm-edge

### osm-edge Configuration

By default, the control plane components are installed into a Kubernetes Namespace called `osm-system` and the control plane is given a unique identifier attribute `mesh-name` defaulted to `osm`.
During installation, the Namespace and mesh-name can be configured through flags when using the `osm` CLI or by editing the values file when using the `helm` CLI.

The `mesh-name` is a unique identifier assigned to an osm-controller instance during install to identify and manage a mesh instance.

The `mesh-name` should follow [RFC 1123](https://tools.ietf.org/html/rfc1123) DNS Label constraints. The `mesh-name` must:

- contain at most 63 characters
- contain only lowercase alphanumeric characters or '-'
- start with an alphanumeric character
- end with an alphanumeric character

### Using the osm-edge CLI

Use the `osm` CLI to install the osm-edge control plane on to a Kubernetes cluster.

Run `osm install`.

```console
# Install osm control plane components
$ osm install
osm-edge installed successfully in namespace [osm-system] with mesh name [osm]
```

Run `osm install --help` for more options.
