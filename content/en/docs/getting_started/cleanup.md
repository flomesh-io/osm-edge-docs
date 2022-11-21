---
title: "Uninstall osm-edge"
description: "Uninstall osm-edge and the bookstore applications"
type: docs
weight: 6
---

# Uninstall osm-edge

The articles in the getting started section outline installing osm-edge and sample applications. To uninstall all of these resources, delete the sample application and related SMI resources and uninstall the osm-edge control plane and cluster-wide osm-edge resources.

## Delete the sample applications

To clean up the sample applications and related SMI resources, delete their namespaces. For example:

```bash
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

## Uninstall osm-edge control plane

To uninstall osm-edge control plane, use `osm unistall mesh`.

```bash
osm uninstall mesh
```

## Uninstall osm-edge cluster-wide resources

To uninstall osm-edge cluster-wide resources, use `osm uninstall mesh --delete-cluster-wide-resources`.

```bash
osm uninstall mesh --delete-cluster-wide-resources
```

For more details about uninstalling osm-edge, see the [uninstallation guide](docs/guides/uninstall/).
