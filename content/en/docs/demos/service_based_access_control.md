---
title: "Service-based access control"
description: "Managing access to services"
type: docs
weight: 18
---

This guide demonstrates an access control mechanism applied at Service level to services within the mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have osm-edge installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.


## Demo

deploy the sample services `httpbin` and `curl`.

```shell
#Mock target service
kubectl create namespace httpbin
osm namespace add httpbin
kubectl apply -n httpbin -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/samples/httpbin/httpbin.yaml

#Mock external service
kubectl create namespace curl
kubectl apply -n curl -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/samples/curl/curl.yaml

#Wait for the dependent POD to start normally
kubectl wait --for=condition=ready pod -n httpbin -l app=httpbin --timeout=180s
kubectl wait --for=condition=ready pod -n curl -l app=curl --timeout=180s
```

At this point, we send a request from service  `curl` to target service `httpbin` by executing the following command.

```shell
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -- curl -sI http://httpbin.httpbin:14001/get
command terminated with exit code 56
```

The access fails because by default the services outside the mesh cannot access the services inside the mesh and we need to apply an access control policy.

Before applying the policy, you need to enable the access control feature, which is disabled by default.

```shell
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableAccessControlPolicy":true}}}' --type= merge
```

## Plaintext transfer

Data can be transferred in plaintext or with two-way TLS encryption. Plaintext transfer is relatively simple, so let's demonstrate the plaintext transfer scenario first.

### Service-based access control

First, create a `Service` for service `curl`.

```shell
kubectl apply -n curl -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: curl
  labels:
    app: curl
    service: curl
spec:
  ports:
    - name: http
      port: 80
  selector:
    app: curl
EOF
```

Next, create an access control policy with the source `Service` `curl` and the target service `httpbin`.

```shell
kubectl apply -f - <<EOF
kind: AccessControl
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
    namespace: curl
    name: curl
EOF
```
Execute the command again to send the authentication request, and you can see that this time an `HTTP 200` response is received.

```shell
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -- curl -sI http://httpbin.httpbin:14001/get
HTTP/1.1 200 OK
server: gunicorn/19.9.0
date: Mon, 07 Nov 2022 08:47:55 GMT
content-type: application/json
content-length: 267
access-control-allow-origin: *
access-control-allow-credentials: true
osm-stats-namespace: httpbin
osm-stats-kind: Deployment
osm-stats-name: httpbin
osm-stats-pod: httpbin-69dc7d545c-qphrh
connection: keep-alive
```

Remember to execute `kubectl delete accesscontrol httpbin -n httpbin` to clean up the policy.

The previous ones we used were plaintext transfers, next we look at encrypted transfers.

## Encrypted transfers

The default access policy certificate feature is off, turn it on by executing the following command.

```shell
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableAccessCertPolicy":true}}}' --type=merge
```

Create `AccessCert` for the access source to assign a certificate for data encryption. The controller will store the certificate information in `Secret` `curl-mtls-secret` under the namespace `curl`, and here also assign SAN `curl.curl.cluster.local` for the access source.

```shell
kubectl apply -f - <<EOF
kind: AccessCert
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: curl-mtls-cert
  namespace: httpbin
spec:
  subjectAltNames:
  - curl.curl.cluster.local
  secret:
    name: curl-mtls-secret
    namespace: curl
EOF
```

Redeploy `curl` and mount the system-assigned `Secret` to the pod.

```shell
kubectl apply -n curl -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      serviceAccountName: curl
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - image: curlimages/curl
        imagePullPolicy: IfNotPresent
        name: curl
        command: ["sleep", "365d"]
        volumeMounts:
        - name: curl-mtls-secret
          mountPath: "/certs"
          readOnly: true
      volumes:
        - name: curl-mtls-secret
          secret:
            secretName: curl-mtls-secret
EOF
```