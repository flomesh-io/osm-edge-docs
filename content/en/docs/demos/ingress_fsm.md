---
title: "Ingress with FSM"
description: "HTTP ingress implemented by the FSM Ingress controller"
type: docs
weight: 10
draft: false
---

osm-edge can optionally use the [FSM](git@github.com:flomesh-io/fsm.git) ingress controller and Pipy-based edge proxies to route external traffic to the Service Mesh backend. This guide demonstrates how to configure HTTP ingress for services managed by the osm-edge service mesh.

## Prerequisites

- Kubernetes cluster version {{< param min_k8s_version >}} or higher.
- Interact with the API server using `kubectl`.
- osm-edge is not installed, and must be removed if it is installed.
- Installed `osm` or `Helm 3` command line tool for installing osm-edge and FSM.
- osm-edge version >= v1.1.0.

## Demo

First, install osm-edge and fsm under the `osm-system` namespace, and name the grid `osm`.

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge will be installed
export osm_mesh_name=osm # Replace osm with the desired osm-edge mesh name
```

Using the `osm` command line tool.

```bash
osm install --set fsm.enabled=true \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace"
```

Using ``Helm`` to install.

```bash
helm install "$osm_mesh_name" osm-edge --repo https://flomesh-io.github.io/osm-edge \
    --set fsm.enabled=true
```

In order to authorize clients by restricting access to backend traffic, we will configure IngressBackend so that only ingress traffic from the `ingress-pipy-controller` endpoint can be routed to the backend service. In order to discover the `ingress-pipy-controller` endpoint, we need the osm-edge controller and the corresponding namespace to monitor it. However, to ensure that the FSM functions properly, it cannot be injected with a Pipy sidecar.

```bash
kubectl label namespace "$osm_namespace" openservicemesh.io/monitored-by="$osm_mesh_name"
```

Save the external IP address and port of the entry gateway, which will be used later to test access to the backend application.

```bash
export ingress_host="$(kubectl -n "$osm_namespace" get service fsm-ingress-pipy-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
export ingress_port="$(kubectl -n "$osm_namespace" get service fsm-ingress-pipy-controller -o jsonpath='{.spec.ports[?(@.name=="http")].port}')"
```

The next step is to deploy the sample `httpbin` service.

```bash
# Create a namespace
kubectl create ns httpbin

# Add the namespace to the mesh
osm namespace add httpbin

# Deploy the application
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

Ensure that the `httpbin` service and pod are up and running properly by

```console
$ kubectl get pods -n httpbin
NAME READY STATUS RESTARTS AGE
httpbin-74677b7df7-zzlm2 2/2 Running 0 11h

$ kubectl get svc -n httpbin
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
httpbin ClusterIP 10.0.22.196 <none> 14001/TCP 11h
```

### HTTP Ingress

Next, create the necessary HTTPProxy and IngressBackend configurations to allow external clients to access port `14001` of the `httpbin` service under the `httpbin` namespace. Because TLS is not used, the link from the fsm entry gateway to the `httpbin` backend pod is not encrypted.

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
spec:
  ingressClassName: pipy
  rules:
  - host: httpbin.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: http
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: fsm-ingress-pipy-controller
EOF
```

Now we expect external clients to have access to the `httpbin` service, with the `HOST` request header of the HTTP request being `httpbin.org`.

```console
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
server: gunicorn/19.9.0
date: Tue, 05 Jul 2022 07:34:11 GMT
content-type: application/json
content-length: 241
access-control-allow-origin: *
access-control-allow-credentials: true
connection: keep-alive
```