---
title: "Fault Injection"
description: "Adding Fault Injection functionality to osm-edge sidecar via Plugins"
weight: 3
---

Fault injection testing is a software testing technique that intentionally introduces errors into a system to verify its ability to handle and bounce back from error conditions. This testing method is usually performed before deployment to identify any possible faults that may have arisen during production. This demo demonstrates on how to implement Fault Injection functionality via a plugin.

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
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/plugins/fault-injection.yaml
```

### Setting up plugin-chain

```bash
kubectl apply -f - <<EOF
kind: PluginChain
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: http-fault-injection-chain
  namespace: pipy
spec:
  chains:
    - name: inbound-http
      plugins:
        - http-fault-injection
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

In below configuration, we need to have either of `delay` or `abort`

```bash
kubectl apply -f - <<EOF
kind: PluginConfig
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: http-fault-injection-config
  namespace: pipy
spec:
  config:
    delay:
      percentage:
        value: 0.5
      fixedDelay: 5s
    abort:
      percentage:
        value: 0.5
      httpStatus: 400
  plugin: http-fault-injection
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

date;  kubectl exec ${curl_client} -n curl -c curl -- curl -ksi http://pipy-ok.pipy:8080 ;  echo "";  date
```

Run the above command a few times, and you will see that after a few more visits, and there is  approximately a 50% chance of receiving the following result (HTTP status code 400, with a delay of 5 seconds).

```bash
Thu Mar 30 06:47:58 UTC 2023
HTTP/1.1 400 Bad Request
content-length: 0
connection: keep-alive


Thu Mar 30 06:48:04 UTC 2023
```


