---
title: "Uninstall the osm-edge Control Plane and Components"
description: "Uninstall"
weight: 4
---

# Uninstall the osm-edge Control Plane and Components

This guide describes how to uninstall osm-edge from a Kubernetes cluster. This guide assumes there is a single osm-edge control plane (mesh) running. If there are multiple meshes in a cluster, repeat the process described for each control plane in the cluster before uninstalling any cluster wide resources at the end of the guide. Taking into consideration both the control plane and dataplane, this guide aims to walk through uninstalling all remnants of osm-edge with minimal downtime.

## Prerequisites

- Kubernetes cluster with osm-edge installed
- The `kubectl` CLI
- The [`osm`](/docs/guides/operating/install/#set-up-the-osm-cli) CLI or the Helm 3 CLI

## Remove Pipy Sidecars from Application Pods and Pipy Secrets

The first step to uninstalling osm-edge is to remove the Pipy sidecar containers from application pods. The sidecar containers enforce traffic policies. Without them, traffic will flow to and from Pods according in accordance with default Kubernetes networking unless there are [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) applied.

osm-edge Pipy sidecars and related secrets will be removed in the following steps:

1. [Disable automatic sidecar injection](#disable-automatic-sidecar-injection)
1. [Restart pods](#restart-pods)

### Disable Automatic Sidecar Injection

osm-edge Automatic Sidecar Injection is most commonly enabled by adding namespaces to the mesh via the `osm` CLI. Use the `osm` CLI to see which
namespaces have sidecar injection enabled. If there are multiple control planes installed, be sure to specify the `--mesh-name` flag.

View namespaces in a mesh:

```console
$ osm namespace list --mesh-name=<mesh-name>
NAMESPACE          MESH           SIDECAR-INJECTION
<namespace1>       <mesh-name>    enabled
<namespace2>       <mesh-name>    enabled
```

Remove each namespace from the mesh:

```console
$ osm namespace remove <namespace> --mesh-name=<mesh-name>
Namespace [<namespace>] successfully removed from mesh [<mesh-name>]
```

This will remove the `openservicemesh.io/sidecar-injection: enabled` annotation and `openservicemesh.io/monitored-by: <mesh name>` label from the namespace. 

Alternatively, if sidecar injection is enabled via annotations on pods instead of per namespace, please modify the pod or deployment spec to remove the sidecar injection annotation.

### Restart Pods

Restart all pods running with a sidecar:

```console
# If pods are running as part of a Kubernetes deployment
# Can use this strategy for daemonset as well
$ kubectl rollout restart deployment <deployment-name> -n <namespace>

# If pod is running standalone (not part of a deployment or replica set)
$ kubectl delete pod <pod-name> -n namespace
$ k apply -f <pod-spec> # if pod is not restarted as part of replicaset
```

Now, there should be no osm-edge Pipy sidecar containers running as part of the applications that were once part of the mesh. Traffic is no
longer managed by the osm-edge control plane with the `mesh-name` used above. During this process, your applications may experience some downtime
as all the Pods are restarting.

## Uninstall osm-edge Control Plane and Remove User Provided Resources

The osm-edge control plane and related components will be uninstalled in the following steps:


1. [Uninstall the osm-edge control plane](#uninstall-the-osm-edge-control-plane)
2. [Remove User Provided Resources](#remove-user-provided-resources)
3. [Delete osm-edge Namespace](#delete-osm-edge-namespace)
4. [Removal of osm-edge Cluster Wide Resources](#removal-of-osm-edge-cluster-wide-resources)


### Uninstall the osm-edge control plane

Use the `osm` CLI to uninstall the osm-edge control plane from a Kubernetes cluster. The following step will remove:

1. osm-edge controller resources (deployment, service, mesh config, and RBAC)
1. Prometheus, Grafana, Jaeger, and Fluent Bit resources installed by osm-edge
1. Mutating webhook and validating webhook
1. The conversion webhook fields patched by osm-edge to the CRDs installed/required by osm-edge: [CRDs for osm-edge](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/cmd/osm-bootstrap/crds) will be unpatched. To delete cluster wide resources refer to [Removal of osm-edge Cluster Wide Resources](#removal-of-osm-cluster-wide-resources) for more details.

Run `osm uninstall mesh`:

```console
# Uninstall osm control plane components
$ osm uninstall mesh --mesh-name=<mesh-name>
Uninstall osm-edge [mesh name: <mesh-name>] ? [y/n]: y
osm-edge [mesh name: <mesh-name>] uninstalled
```

Run `osm uninstall mesh --help` for more options.

Alternatively, if you used Helm to install the control plane, run the following `helm uninstall` command:

```console
$ helm uninstall <mesh name> --namespace <osm namespace>
```

Run `helm uninstall --help` for more options.

### Remove User Provided Resources

If any resources were provided or created for osm-edge at install time, they can be deleted at this point.

For example, if [Hashicorp Vault](/docs/guides/certificates/#installing-hashi-vault) was deployed for the sole purpose of managing certificates for osm-edge, all related resources can be deleted.

### Delete osm-edge Namespace

When installing a mesh, the `osm` CLI creates the namespace the control plane is installed into if it does not already exist. However, when uninstalling the same mesh, the namespace it lives in does not automatically get deleted by the `osm` CLI. This behavior occurs because
there may be resources a user created in the namespace that they may not want automatically deleted.

If the namespace was only used for osm-edge and there is nothing that needs to be kept around, the namespace can be deleted at the time of uninstall or later using the following command.

```console
$ osm uninstall mesh --delete-namespace
```

> Warning: Only delete the namespace if resources in the namespace are no longer needed. For example, if osm was installed in `kube-system`, deleting the namespace may delete important cluster resources and may have unintended consequences.


### Removal of osm-edge Cluster Wide Resources

On installation osm-edge ensures that all the CRDs mentioned [here](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/cmd/osm-bootstrap/crds) exist in the cluster at install time. During installation, if they are not already installed, the `osm-bootstrap` pod will install them before the rest of the control plane components are running. This is the same behavior when using the Helm charts to install osm-edge as well. 

Uninstalling the mesh in both unmanaged and managed environments:
1. removes osm-edge control plane components, including control plane pods
2. removes/un-patches the conversion webhook fields from all the CRDs (which osm-edge adds to support multiple CR versions)

leaving behind certain osm-edge resources to prevent unintended consequences for the cluster after uninstalling osm-edge.The resources that are left behind will depend on whether osm-edge was uninstalled from a managed or unmanaged cluster environment.

When uninstalling osm-edge, both the `osm uninstall mesh` command and Helm uninstallation will not delete any osm-edge or SMI CRD in any cluster environment (managed and unmanaged) for primarily two reasons:
1. CRDs are cluster-wide resources and may be used by other service meshes or resources running in the same cluster
2. deletion of a CRD will cause all custom resources corresponding to that CRD to also be deleted

To remove cluster wide resources that osm-edge installs (i.e. the meshconfig, secrets, osm-edge CRDs, SMI CRDs, and webhook configurations), the following command can be run during or after osm-edge's uninstillation.

```bash
osm uninstall mesh --delete-cluster-wide-resources
```

> Warning: Deletion of a CRD will cause all custom resources corresponding to that CRD to also be deleted.

To troubleshoot osm-edge uninstallation, refer to the [uninstall troubleshooting section](/docs/guides/troubleshooting/uninstall/)
