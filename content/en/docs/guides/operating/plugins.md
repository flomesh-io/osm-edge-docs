---
title: "Extending osm-edge"
description: "How to extend osm-edge service mesh without re-compiling it"
type: docs
weight: 15
---

# Extending osm-edge with `Plugin` Interface

In the latest 1.3.0 version of Flomesh service mesh osm-edge, we have introduced a significant feature: `Plugin`. This feature aims to provide developers with a way to extend the functionality of the service mesh without changing the osm-edge itself.

Nowadays, service mesh seems to be developing in two directions. One is like `Istio`, which provides a lot of ready-to-use functions and is very rich in features. The other like `Linkerd`, Flomesh `osm-edge`, and others that uphold the principle of simplicity and provide a minimum functional set that meets the user's needs. There is no superiority or inferiority between the two: the former is rich in features but inevitably has the additional overhead of proxy, not only in resource consumption but also in the cost of learning and maintenance; the latter is easy to learn and use, consumes fewer resources, but the provided functions might not be enough for the immediate need of user desired functionality.

It is not difficult to imagine that the ideal solution is the low cost of the minimum functional set + the flexibility of scalability. The core of the service mesh is in the data plane, and the flexibility of scalability requires a high demand for the physique of the sidecar proxy. This is also why the Flomesh service mesh chose programmable proxy [Pipy](https://flomesh.io/pipy) as the sidecar proxy.

Pipy is a programmable network proxy for cloud, edge, and IoT. It is flexible, fast, small, programmable, and open-source. The modular design of Pipy provides a large number of reusable filters that can be assembled into pipelines to process network data. Pipy provides a set of api and small usable filters to achieve business objectives while hiding the underlying details. Additionally, Pipy scripts (programming code that implements functional logic) can be dynamically delivered to Pipy instances over the network, enabling the proxy to be extended with new features without the need for compilation or restart.

## Flomesh osm-edge extension solution

osm-edge provides three new CRDs for extensibility:

- `Plugin`: The plugin contains the code logic for the new functionality. The default functions provided by osm-edge are also available as plugins, but not in the form of a `Plugin` resource. These plugins can be adjusted through the Helm values file when installing osm-edge. For more information, refer to the built-in plugin list in the Helm [values.yaml](https://github.com/flomesh-io/osm-edge/blob/45b05bd39dc0e8d1c28460622a4be2f92abdf28f/charts/osm/values.yaml#L84) file.
- `PluginChain`: The plugin chain is the execution of plugins in sequence. The system provides four plugin chains: `inbound-tcp`, `inbound-http`, `outbound-tcp`, `outbound-http`. They correspond to the OSI layer-4 and layer-7 processing stages of inbound and outbound traffic, respectively.
- `PluginConfig`: The plugin configuration provides the configuration required for the plugin logic to run, which will be sent to the osm-edge sidecar proxy in JSON format.

For detailed information on plugin CRDs, refer to the [Plugin API document](/docs/api_reference/plugin/).

## Demo

For a simple demonstration of how to extend osm-edge via `Plugins`, refer to below demo:

* [Adding Identity and Access Management functionality](/docs/demos/plugin_iam_demo)