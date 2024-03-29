---
title: "Application Container Lifecycle"
description: "Troubleshooting application container lifecycle"
type: docs
weight: 1
---

# Troubleshooting Application Container Lifecycle

Since osm-edge injects application pods that are a part of the service mesh with a long-running sidecar proxy and sets up traffic redirection rules to route all traffic to/from pods via the sidecar proxy, in some circumstances existing application containers might not startup or shutdown as expected.

## When the application container depends on network connectivity at startup

Application containers that depend on network connectivity at startup are likely to experience issues once the Pipy sidecar proxy container and the `osm-init` init container are injected into the application pod by osm-edge. This is because upon sidecar injection, all TCP based network traffic from application containers are routed to the sidecar proxy and subject to service mesh traffic policies. This implies that for application traffic to be routed as it would without the sidecar proxy container injected, osm-edge controller must first program the sidecar proxy on the application pod to allow such traffic. Without the Pipy sidecar proxy being configured, all traffic from application containers will be dropped.

When osm-edge is configured with permissive traffic policy mode enabled, osm-edge will program wildcard traffic policy rules on the Pipy sidecar proxy to allow every pod to access all services that are a part of the mesh. When osm-edge is configured with SMI traffic policy mode enabled, explicit SMI policies must be configured to enable communication between applications in the mesh.

Regardless of the traffic policy mode, application containers that depend on network connectivity at startup can experience problems starting up if they are not resilient to delays in the network being ready. With the Pipy proxy sidecar injected, the network is deemed ready only when the sidecar proxy has been programmed by osm-edge controller to allow application traffic to flow through the network.

It is recommended that application containers be resilient enough to the initial bootstrapping phase of the Pipy proxy sidecar in the application pod.

It is important to note that the [container's restart policy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) also influences the startup of application containers. If an application container's startup policy is set to `Never` and it depends on network connectivity to be ready at startup time, it is possible the container fails to access the network until the Pipy proxy sidecar is ready to allow the application container access to the network, thereby resulting in the application container to exit and never recover from a failed startup. For this reason, it is recommended not to use a container restart policy of `Never` if your application container depends on network connectivity at startup.

## When the application container is designed to run to completion

Application containers, like those running in [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/), that are meant to perform a certain set of tasks and then exit may not appear to have finished based on the Job's status. This status is  expected and is due to the Envoy sidecar container running indefinitely. Update your pods controlled by Jobs to use the Envoy sidecar's `/quitquitquit` HTTP endpoint to signal for Envoy to exit when your pod has completed its tasks.

 For Jobs driven by shell scripts, this may look something like

```
trap 'curl localhost:15000/quitquitquit -X POST' EXIT
```

Or if `curl` isn't available but `busybox` is,

```
trap 'nc 127.0.0.1 15000 -e echo -ne "POST /quitquitquit HTTP/1.1\nHost: 127.0.0.1:15000\n\n"' EXIT
```

### Related issues (work in progress)

- [osm-edge issue 2316](https://github.com/flomesh-io/osm-edge/issues/2316): Defer startup of application containers till the Pipy proxy sidecar is ready
- [Kubernetes issue 65502](https://github.com/kubernetes/kubernetes/issues/65502): Support startup dependencies between containers on the same pod

