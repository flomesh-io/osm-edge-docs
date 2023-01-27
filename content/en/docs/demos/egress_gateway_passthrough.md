---
title: "Egress Gateway Passthrough to Unknown Destinations"
description: "Accessing external services via Egress Gateway without Egress policies"
type: docs
weight: 17
---

This guide demonstrates a client within the service mesh accessing destinations external to the mesh via egress gateway using osm-edge's Egress capability to passthrough traffic to unknown destinations without an Egress policy.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have osm-edge installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- Have `helm` CLI avaiable to install fsm.

## Egress Gateway passthrough demo

1. Deploy egress gateway via fsm.
    
    ```bash
    helm repo add fsm https://flomesh-io.github.io/fsm
    helm repo update
    helm install --namespace fsm --create-namespace --set fsm.version=0.2.0 --set fsm.egressGateway.enabled=true fsm fsm/fsm
    ```

2. Declear egress gateway.
    
    ```bash
    kubectl apply -f - <<EOF
    kind: EgressGateway
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
    name: global-egress-gateway
    namespace: curl
    spec:
    global:
        - service: fsm-egress-gateway
        namespace: fsm
    EOF
    ```

3. Enable global egress passthrough if not enabled:
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge
    ```

4. Deploy the `curl` client into the `curl` namespace after enrolling its namespace to the mesh.

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -n curl -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml
    ```

    Confirm the `curl` client pod is up and running.

    ```bash
    kubectl get pods -n curl 
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-7bb5845476-8s9kv   2/2     Running   0          29s
    ```

5. Confirm the `curl` client is able to make successful HTTP requests to the `httpbin.org` website on port `80`.

    ```bash
    kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://httpbin.org:80/get
    HTTP/1.1 200 OK
    Date: Fri, 27 Jan 2023 22:27:53 GMT
    Content-Type: application/json
    Content-Length: 258
    Connection: keep-alive
    Server: gunicorn/19.9.0
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    ```

    A `200 OK` response indicates the HTTP request from the `curl` client to the `httpbin.org` website was successful.

6. Confirm the HTTP requests fail when mesh-wide egress is disabled.

    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
    ```

    ```bash
    kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    ```