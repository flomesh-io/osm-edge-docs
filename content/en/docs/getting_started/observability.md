---
title: "Configure Observability with Prometheus and Grafana"
description: "Use osm-edge's observability integrations with Prometheus and Grafana to inspect the traffic between the bookstore applications"
type: docs
weight: 5
---

# Configure Observability with Prometheus and Grafana

The following article shows you how to install osm-edge with automatic provisioning of the Prometheus and Grafana stack for observability and monitoring. For an example using a bring your own (BYO) Prometheus and Grafana stack on your cluster with osm-edge, see the [Integrate osm-edge with Prometheus and Grafana](/docs/demos/prometheus_grafana/) demo.

The configuration created in this article should not be used in production environments. For production-grade deployments, see [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md) and [Deploy Grafana in Kubernetes](https://grafana.com/docs/grafana/latest/installation/kubernetes/).


## Install osm-edge with Prometheus and Grafana

On `osm install`, a Prometheus and/or Grafana instance can be automatically provisioned with the default osm-edge configuration.
```bash
 osm install --set=osm.deployPrometheus=true \
             --set=osm.deployGrafana=true
```
More information on observability can be found in the [Observability Guide](/docs/guides/observability).

## Prometheus

When configured with the `--set=osm.deployPrometheus=true` flag, osm-edge installation will deploy a Prometheus instance to scrape the sidecar and osm-edge control plane's metrics endpoints. The [scraping configuration file](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/templates/prometheus-configmap.yaml) defines the default Prometheus behavior and the set of metrics collected by osm-edge.

## Grafana

osm-edge can be configured to deploy a [Grafana](https://grafana.com/grafana/) instance using the `--set=osm.deployGrafana=true` flag in `osm install`. osm-edge provides pre-configured dashboards that are documented in the [osm-edge Grafana dashboards](/docs/guides/observability/metrics/#osm-grafana-dashboards) section of the Observability Guide.

## Enable Metrics Scraping

Metrics can be enabled at the namespace scope using the `osm metrics` command. By default, osm-edge **does not** configure metrics scraping for pods in the mesh. 
```bash
osm metrics enable --namespace test
osm metrics enable --namespace "test1, test2"

```
> Note: The namespace that you are enabling for metrics scraping must already be a part of the mesh.

## Inspect Dashboards

The osm-edge Grafana dashboards can be viewed with the following command:

```bash
osm dashboard
```

Navigate to http://localhost:3000 to access the Grafana dashboards. The default user name is `admin` and the default password is `admin`. On the Grafana homepage click on the **Home** icon, you will see a folder containing dashboards for both osm-edge Control Plane and osm-edge Data Plane.

## Next Steps

[Cleanup sample applications and uninstall osm-edge](/docs/getting_started/cleanup/).