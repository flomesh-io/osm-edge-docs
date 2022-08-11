---
title: "Ingress with Traefik"
description: "Kubernetes ingress with Traefik Ingress Controller"
type: docs
weight: 12
draft: false
---

This article demonstrates how to use [Traefik Ingress](https://doc.traefik.io/traefik/providers/kubernetes-ingress/) to access the services hosted by the osm-edge service mesh.

## Prerequisites 

* Kubernetes cluster version v1.19.0 or higher.
* Use kubectl to interact with the API server.
* osm-edge is not installed, and must be removed first if installed.
* osm cli is installed to install osm-edge.
* Helm 3 command line tool is installed for traefik installation.
* osm-edge version >= v1.1.0.

## Demo

### Install Traefik

```shell
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install traefik traefik/traefik -n traefik --create-namespace
```

Verify that the pod is up and running.

```shell
kubectl get po -n traefik
NAME READY STATUS RESTARTS AGE
traefik-69fb598d54-9v9vf 1/1 Running 0 24s
```

Retrieve and store external IP address and port of the entry gateway to environment variables, which will be used later to access the application.

```
export ingress_host="$(kubectl -n traefik get service traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
export ingress_port="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[? (@.name=="web")].port}')"
```

### Install osm-edge

```shell
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true
```

Confirm that the pod is up and running.

```shell
kubectl get po -n osm-system
NAME READY STATUS RESTARTS AGE
osm-bootstrap-6477f776cc-d5r89 1/1 Running 0 2m51s
osm-injector-5696694cf6-7kvpt 1/1 Running 0 2m51s
osm-controller-86d68c557b-tvgtm 2/2 Running 0 2m51s
```

### Deploy sample service

```shell
kubectl create ns httpbin
osm namespace add httpbin
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

Confirm that the service has been created and the pod is up and running.

```shell
kubectl get svc -n httpbin
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
httpbin ClusterIP 10.43.51.114 <none> 14001/TCP 9s

kubectl get po -n httpbin
NAME READY STATUS RESTARTS AGE
httpbin-69dc7d545c-bsjxx 2/2 Running 0 77s
```

### HTTP Ingress

Next, create an ingress to expose the `14001` port of the `httpbin` service under the `httpbin` namespace.

```shell
kubectl apply -f - <<EOF
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: httpbin
  namespace: httpbin
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
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
EOF
```

Using the entry gateway address and port saved earlier to access the service, you should receive a response of `502` at this point. This is normal, because you still need to create `IngressBackend` to allow the entry gateway to access the `httpbin` service.

```shell
curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 502 Bad Gateway
Date: Tue, 09 Aug 2022 13:17:11 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8
```

Execute the following command to create ``IngressBackend``.

```shell
kubectl apply -f - <<EOF
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
    namespace: traefik
    name: traefik
EOF
```

Now, re-visit `httpbin` and you will be able to access it successfully.

```shell
curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Content-Length: 338
Content-Type: application/json
Date: Tue, 09 Aug 2022 13:17:41 GMT
Osm-Stats-Kind: Deployment
Osm-Stats-Name: httpbin
Osm-Stats-Namespace: httpbin
Osm-Stats-Pod: httpbin-69dc7d545c-bsjxx
Server: gunicorn/19.9.0
```