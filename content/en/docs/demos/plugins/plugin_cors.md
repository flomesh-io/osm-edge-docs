---
title: "Cross-Origin Resource Sharing (CORS)"
description: "Adding CORS functionality to osm-edge sidecar via Plugins"
weight: 5
---

CORS stands for Cross-Origin Resource Sharing. It is a security feature implemented by web browsers to prevent web pages from making requests to a different domain than the one that served the web page.

The same-origin policy is a security feature that allows web pages to access resources only from the same origin, which includes the same domain, protocol, and port number. This policy is designed to protect users from malicious scripts that can steal sensitive data from other websites.

CORS allows web pages to make cross-origin requests by adding specific headers to the HTTP response from the server. The headers indicate which domains are allowed to make cross-origin requests, and what type of requests are allowed.

However, configuring CORS can be challenging, especially when dealing with complex web applications that involve multiple domains and servers. One way to simplify the CORS configuration is to use a proxy server.

A proxy server acts as an intermediary between the web application and the server. The web application sends requests to the proxy server, which then forwards the requests to the server. The server responds to the proxy server, which then sends the response back to the web application.

By using a proxy server, you can configure the CORS headers on the proxy server instead of configuring them on the server that serves the web page. This way, the web application can make cross-origin requests without violating the same-origin policy.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have osm-edge installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
 
## Deploy demo services

```bash
kubectl create namespace curl
osm namespace add curl
kubectl apply -n curl -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/plugins/curl.yaml

kubectl create namespace pipy
osm namespace add pipy
kubectl apply -n pipy -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/plugins/pipy-ok.pipy.yaml

# Wait for pods to be up and ready

sleep 2
kubectl wait --for=condition=ready pod -n curl -l app=curl --timeout=180s
kubectl wait --for=condition=ready pod -n pipy -l app=pipy-ok -l version=v1 --timeout=180s
kubectl wait --for=condition=ready pod -n pipy -l app=pipy-ok -l version=v2 --timeout=180s
```

### Enable plugin policy mode

```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enablePluginPolicy":true}}}' --type=merge
```

### Declaring a plugin

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/plugins/cors.yaml
```

### Setting up plugin-chain

```bash
kubectl apply -f - <<EOF
kind: PluginChain
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: cors-policy-chain
  namespace: pipy
spec:
  chains:
    - name: inbound-http
      plugins:
        - cors-policy
  selectors:
    podSelector:
      matchLabels:
        app: pipy-ok
      matchExpressions:
        - key: app
          operator: In
          values: ["pipy-ok"]
    namespaceSelector:
      matchExpressions:
        - key: openservicemesh.io/monitored-by
          operator: In
          values: ["osm"]
EOF
```

### Setting up plugin configuration

```bash
kubectl apply -f - <<EOF
kind: PluginConfig
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: cors-policy-config
  namespace: pipy
spec:
  config:
    allowCredentials: true
    allowHeaders:
    - X-Foo-Bar-1
    allowMethods:
    - POST
    - GET
    - PATCH
    - DELETE
    allowOrigins:
    - regex: http.*://www.test.cn
    - exact: http://www.aaa.com
    - prefix: http://www.bbb.com
    exposeHeaders:
    - Content-Encoding
    - Kuma-Revision
    maxAge: 24h
  plugin: cors-policy
  destinationRefs:
    - kind: Service
      name: pipy-ok
      namespace: pipy
EOF
```

## Test

Use the below command to perform a test

```bash
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n curl -c curl -- curl -ksi http://pipy-ok.pipy:8080 -H "Origin: http://www.bbb.com"
```

You will see response similar to:

```console
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-expose-headers: Content-Encoding,Kuma-Revision
access-control-allow-origin: http://www.bbb.com
content-length: 20
connection: keep-alive

Hi, I am PIPY-OK v1!
```


Run another command to perform a test

```bash
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n curl -c curl -- curl -ksi http://pipy-ok.pipy:8080 -H "Origin: http://www.bbb.com" -X OPTIONS
```

You will see response similar to:

```console
HTTP/1.1 200 OK
access-control-allow-origin: http://www.bbb.com
access-control-allow-credentials: true
access-control-expose-headers: Content-Encoding,Kuma-Revision
access-control-allow-methods: POST,GET,PATCH,DELETE
access-control-allow-headers: X-Foo-Bar-1
access-control-max-age: 86400
content-length: 0
connection: keep-alive
```