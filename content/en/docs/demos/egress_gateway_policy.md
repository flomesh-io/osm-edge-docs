---
title: "Egress Gateway Policy"
description: "Accessing external services via Egress Gateway using Egress policies"
weight: 18
---

This guide demonstrates a client within the service mesh accessing destinations external to the mesh via egress gateway using osm-edge's Egress policy API.

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
2. Declare egress gateway.
    
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
   

3. Disable global egress passthrough to enable egress policy if not disabled:
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
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

5. Confirm the `curl` client is unable make the HTTP request `http://httpbin.org:80/get` to the `httpbin.org` website on port `80`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    ```

6. Apply an Egress policy to allow the `curl` client's ServiceAccount to access the `httpbin.org` website on port `80` serving the `http` protocol.
    
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-80
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 80
        protocol: http
    EOF

7. Confirm the `curl` client is able to make successful HTTP requests to `http://httpbin.org:80/get`.

    ```bash
    kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    HTTP/1.1 200 OK
    date: Fri, 27 Jan 2023 22:31:46 GMT
    content-type: application/json
    content-length: 314
    server: gunicorn/19.9.0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    connection: keep-alive
    ```