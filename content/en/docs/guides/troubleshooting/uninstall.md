---
title: "Uninstall Troubleshooting"
description: "osm-edge Mesh Uninstall Troubleshooting Guide"
aliases: ["/docs/troubleshooting/uninstall"]
type: docs
---

# osm-edge Mesh Uninstall Troubleshooting Guide

If for any reason, `osm uninstall mesh` (as documented in the [uninstall guide](/docs/guides/uninstall/)) fails, you may manually delete osm-edge resources as detailed below.

Set environment variables for your mesh:
```console
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
export mesh_name=osm # Replace osm with the osm-edge mesh name
export osm_version=<osm version>
export osm_ca_bundle=<osm ca bundle>
```

Delete osm-edge control plane deployments:
```console
kubectl delete deployment -n $osm_namespace osm-bootstrap
kubectl delete deployment -n $osm_namespace osm-controller
kubectl delete deployment -n $osm_namespace osm-injector
```

If osm-edge was installed alongside Prometheus, Grafana, or Jaeger, delete those deployments:
```console
kubectl delete deployment -n $osm_namespace osm-prometheus
kubectl delete deployment -n $osm_namespace osm-grafana
kubectl delete deployment -n $osm_namespace jaeger
```

If osm-edge was installed with the osm-edge Multicluster Gateway, delete it by running the following:
```console
kubectl delete deployment -n $osm_namespace osm-multicluster-gateway
```

Delete osm-edge secrets, the meshconfig, and webhook configurations:
> Warning: Ensure that no resources in the cluster depend on the following resources before proceeding.
```console
kubectl delete secret -n $osm_namespace $osm_ca_bundle mutating-webhook-cert-secret validating-webhook-cert-secret crd-converter-cert-secret
kubectl delete meshconfig -n $osm_namespace osm-mesh-config
kubectl delete mutatingwebhookconfiguration -l app.kubernetes.io/name=openservicemesh.io,app.kubernetes.io/instance=$mesh_name,app.kubernetes.io/version=$osm_version,app=osm-injector
kubectl delete validatingwebhookconfiguration -l app.kubernetes.io/name=openservicemesh.io,app.kubernetes.io/instance=mesh_name,app.kubernetes.io/version=$osm_version,app=osm-controller
```

To delete osm-edge and SMI CRDs from the cluster, run the following.
> Warning: Deletion of a CRD will cause all custom resources corresponding to that CRD to also be deleted.
```console
kubectl delete crd meshconfigs.config.openservicemesh.io
kubectl delete crd multiclusterservices.config.openservicemesh.io
kubectl delete crd egresses.policy.openservicemesh.io
kubectl delete crd ingressbackends.policy.openservicemesh.io
kubectl delete crd httproutegroups.specs.smi-spec.io
kubectl delete crd tcproutes.specs.smi-spec.io
kubectl delete crd traffictargets.access.smi-spec.io
kubectl delete crd trafficsplits.split.smi-spec.io
```