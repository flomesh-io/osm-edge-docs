---
title: "Reconciler Guide"
description: "Reconciler Guide"
aliases: ["/docs/reconciler_guide"]
type: docs
weight: 8
---

# Reconciler Guide

This guide describes how to enable the reconciler in osm-edge.

## How the reconciler works

The goal of building a reconciler in osm-edge is to ensure resources required for the correct operation of osm-edge's control plane are in their desired state at all times. Resources that are installed as a part of osm-edge install and have the labels `openservicemesh.io/reconcile: true` and `app.kubernetes.io/name: openservicemesh.io` will be reconciled by the reconciler.

**Note**: The reconciler will not operate as desired if the lables `openservicemesh.io/reconcile: true` and `app.kubernetes.io/name: openservicemesh.io` are modified or deleted on the reconcilable resources.

An update or delete event on the reconcilable resources will trigger the reconciler and it will reconcile the resource back to its desired state. Only metadata changes (excluding a name change) will be permitted on the reconcilable resources.

### Resources reconciled

The resources that osm-edge reconciles are:

- CRDs : The CRDs installed/required by osm-edge [CRDs for osm-edge](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/cmd/osm-bootstrap/crds) will be reconciled. Since osm-edge manages the installation and upgrade of the CRDs it needs, osm-edge will also reconcile them to ensure that their spec, stored and served verions are always in the state that is required by osm-edge.

- MutatingWebhookConfiguration : A MutatingWebhookConfiguration is deployed as a part of osm-edge's control plane to enable automatic sidecar injection. As this is a very critical component for pods joining the mesh, osm-edge reconciles this resource.

- ValidatingWebhookConfiguration : A ValidatingWebhookConfiguration is deployed as a part of osm-edge's control plane to validate various mesh configurations. This resources validates configurations being applied to the mesh, hence osm-edge will reconcile this resource.


## How to install osm-edge with the reconciler

To install osm-edge with the reconciler, use the below command:

```console
$ osm install --set osm.enableReconciler=true
osm-edge installed successfully in namespace [osm-system] with mesh name [osm]
```

