---
title: "Traffic Mirroring"
description: "Adding Traffic Shadowing functionality to osm-edge sidecar via Plugins"
weight: 4
---

Traffic mirroring, also known as shadowing, is a technique used to test new versions of an application in a safe and efficient manner. It involves creating a mirrored service that receives a copy of live traffic for testing and troubleshooting purposes. This approach is especially useful for acceptance testing, as it can help identify issues in advance, before they impact end-users.

One of the key benefits of traffic mirroring is that it occurs outside the primary request path for the main service. This means that end-users are not affected by any changes or issues that may occur during the testing process. As such, traffic mirroring is a powerful and low-risk approach for validating new versions of an application.

By using traffic mirroring, you can get valuable insights into how your application will perform in a live environment, without putting your users at risk. This approach can help you identify and address issues quickly and efficiently, which can ultimately improve the overall performance and reliability of your application.

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
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/plugins/traffic-mirror.yaml
```

### Setting up plugin-chain

```bash
kubectl apply -f - <<EOF
kind: PluginChain
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: traffic-mirror-chain
  namespace: pipy
spec:
  chains:
    - name: outbound-http
      plugins:
        - traffic-mirror
  selectors:
    podSelector:
      matchLabels:
        app: curl
      matchExpressions:
        - key: app
          operator: In
          values: ["curl"]
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
  name: traffic-mirror-config
  namespace: curl
spec:
  config:
    namespace: pipy
    service: pipy-ok-v2
    port: 8080
    percentage:
      value: 1.0
  plugin: traffic-mirror
  destinationRefs:
    - kind: Service
      name: pipy-ok-v1
      namespace: pipy
EOF
```

## Test

Use the below command to perform a test

```bash
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n curl -c curl -- curl -ksi http://pipy-ok-v1.pipy:8080 
```

Accessing service `pipy-ok-v1` should be mirrored to `pipy-ok-v2`, so we should be seeing access logs in both services.

### Testing pipy-ok-v1 logs

```bash
pipy_ok_v1="$(kubectl get pod -n pipy -l app=pipy-ok,version=v1 -o jsonpath='{.items[0].metadata.name}')"
kubectl logs pod/${pipy_ok_v1} -n pipy -c pipy
```

You will see something similar

```console
[2023-03-29 08:41:04 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2023-03-29 08:41:04 +0000] [1] [INFO] Listening at: http://0.0.0.0:8080 (1)
[2023-03-29 08:41:04 +0000] [1] [INFO] Using worker: sync
[2023-03-29 08:41:04 +0000] [14] [INFO] Booting worker with pid: 14
127.0.0.6 - - [29/Mar/2023:08:45:35 +0000] "GET / HTTP/1.1" 200 9593 "-" "curl/7.85.0-DEV"
```

### Testing pipy-ok-v2 logs

```bash
pipy_ok_v2="$(kubectl get pod -n pipy -l app=pipy-ok,version=v2 -o jsonpath='{.items[0].metadata.name}')"
kubectl logs pod/${pipy_ok_v2} -n pipy -c pipy
```

You will see something similar

```console
[2023-03-29 08:41:09 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2023-03-29 08:41:09 +0000] [1] [INFO] Listening at: http://0.0.0.0:8080 (1)
[2023-03-29 08:41:09 +0000] [1] [INFO] Using worker: sync
[2023-03-29 08:41:09 +0000] [15] [INFO] Booting worker with pid: 15
127.0.0.6 - - [29/Mar/2023:08:45:35 +0000] "GET / HTTP/1.1" 200 9593 "-" "curl/7.85.0-DEV"
```