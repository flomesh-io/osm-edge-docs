---
title: "Bi-direction TLS with NginX Ingress"
description: "Configuring different TLS certificates for Ingress and Egress"
weight: 13
---

This guide will demonstrate with multiple scenarios on how to configure different TLS certificates for Nginx Ingress and Egress communication.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have osm-edge installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- Have Kubernetes Nginx Ingress Controller installed. Refer to the [deployment guide](https://kubernetes.github.io/ingress-nginx/deploy/) to install it.

### Deploy demo pods

```bash
#Sample server service
kubectl create namespace egress-server
kubectl apply -n egress-server -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/server.yaml

#Sample middle-ware service
kubectl create namespace egress-middle
osm namespace add egress-middle
kubectl apply -n egress-middle -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/middle.yaml

#Sample client
kubectl create namespace egress-client
kubectl apply -n egress-client -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/client.yaml

#Wait for POD to start properly
kubectl wait --for=condition=ready pod -n egress-server -l app=server --timeout=180s
kubectl wait --for=condition=ready pod -n egress-middle -l app=middle --timeout=180s
kubectl wait --for=condition=ready pod -n egress-client -l app=client --timeout=180s
```

### Scenario#1: HTTP Nginx & HTTP Ingress & mTLS Egress

#### Test commands

Traffic flow:

Client --**http**--> Nginx Ingress Controller

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### Test results

The correct return result is similar to :

```bash
HTTP/1.1 404 Not Found
Date: Sun, 09 Oct 2022 08:36:48 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

#### Setup Ingress Rules

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: middle
            port:
              number: 8080
EOF
```

#### Setup IngressBackend

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of middle service
      protocol: http
  sources:
  - kind: Service
    namespace: ingress-nginx
    name: ingress-nginx-controller
EOF
```

#### Test Commands

Traffic Flow:

Client --**http**--> Nginx Ingress --**http** --> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/1.1 200 OK
Date: Sun, 09 Oct 2022 08:37:18 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 13
Connection: keep-alive
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc

hello world.
```

#### Disable Egress Permissive mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### Enable Egress Policy

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### Create Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/middle.crt -o middle.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/middle.key -o middle.key

kubectl create secret generic -n osm-system egress-middle-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./middle.crt \
  --from-file=tls.key=./middle.key 
```

#### Setup Egress Policy

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: egress-middle
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-middle-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### Test Commands

Traffic Flow:

Client --**http**--> Nginx Ingress --**http**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/time
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/1.1 200 OK
Date: Sun, 09 Oct 2022 08:38:03 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 74
Connection: keep-alive
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc

The current time: 2022-10-09 08:38:03.03610171 +0000 UTC m=+107.351638075
```

This business scenario is tested and the strategy is cleaned up to avoid affecting subsequent tests

```bash
kubectl delete ingress -n egress-middle egress-middle
kubectl delete ingressbackend -n egress-middle egress-middle
kubectl delete egress -n egress-middle server-8443
kubectl delete secrets -n osm-system egress-middle-cert
```

### Scenario#2: HTTP Nginx & mTLS Ingress & mTLS Egress

#### Test Commands

Traffic flow:

Client --**http**--> Nginx Ingress Controller

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/1.1 404 Not Found
Date: Sun, 09 Oct 2022 08:38:27 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

#### Setup Ingress Controller Cert

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":{"secret":{"name":"ingress-controller-cert","namespace":"osm-system"},"subjectAltNames":["ingress-nginx.ingress-nginx.cluster.local"],"validityDuration":"24h"}}}}' --type=merge
```

#### Setup Ingress Rules

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: egress-middle
  namespace: egress-middle
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "middle.egress-middle.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/ingress-controller-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: middle
            port:
              number: 8080
EOF
```

#### Setup IngressBackend Policy

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of middle service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    namespace: ingress-nginx
    name: ingress-nginx-controller
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```

#### Test Commands

Traffic flow:

Client --**http**--> Nginx Ingress --**mtls** --> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/1.1 200 OK
Date: Sun, 09 Oct 2022 08:39:02 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 13
Connection: keep-alive
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc

hello world.
```

#### Disable Egress Permissive mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### Enable Egress Policy

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### Create Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/middle.crt -o middle.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/middle.key -o middle.key

kubectl create secret generic -n osm-system egress-middle-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./middle.crt \
  --from-file=tls.key=./middle.key 
```

#### Setup Egress Policy

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: egress-middle
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-middle-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### Test Commands

Traffic flow:

Client --**http**--> Nginx Ingress --**mtls**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/time
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/1.1 200 OK
Date: Sun, 09 Oct 2022 08:39:42 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 75
Connection: keep-alive
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc

The current time: 2022-10-09 08:39:42.336230168 +0000 UTC m=+206.716581170
```

This business scenario is tested and the strategy is cleaned up to avoid affecting subsequent tests

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":null}}}' --type=merge

kubectl delete ingress -n egress-middle egress-middle
kubectl delete ingressbackend -n egress-middle egress-middle
kubectl delete egress -n egress-middle server-8443
kubectl delete secrets -n osm-system egress-middle-cert
```

### Scenario#3：TLS Nginx & mTLS Ingress & mTLS Egress

#### Test Commands

Traffic flow:

Client --**http**--> Nginx Ingress Controller

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/1.1 404 Not Found
Date: Sun, 09 Oct 2022 08:40:00 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

#### Setup Ingress Controller Cert

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":{"secret":{"name":"ingress-controller-cert","namespace":"osm-system"},"subjectAltNames":["ingress-nginx.ingress-nginx.cluster.local"],"validityDuration":"24h"}}}}' --type=merge
```

#### Create Nginx TLS Secret

```bash
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/nginx.crt -o nginx.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/nginx.key -o nginx.key

kubectl create secret tls -n egress-middle nginx-cert-secret \
  --cert=./nginx.crt \
  --key=./nginx.key 
```

#### Setup Ingress Rules

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: egress-middle
  namespace: egress-middle
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "middle.egress-middle.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/ingress-controller-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: middle
            port:
              number: 8080
  tls:
  - hosts:
    - ingress-nginx-controller.ingress-nginx
    secretName: nginx-cert-secret
EOF
```

#### Setup IngressBackend Policy

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of middle service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    namespace: ingress-nginx
    name: ingress-nginx-controller
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```

#### Test Commands

Traffic flow:

Client --**tls**--> Nginx Ingress --**mtls** --> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -ksi https://ingress-nginx-controller.ingress-nginx/hello --key /certs/client.key --cert /certs/client.crt
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/2 200 
date: Sun, 09 Oct 2022 08:40:40 GMT
content-type: text/plain; charset=utf-8
content-length: 13
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc
strict-transport-security: max-age=15724800; includeSubDomains

hello world.
```

#### Disable Egress Permissive mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### Enable Egress Policy

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### Create Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/middle.crt -o middle.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/middle.key -o middle.key

kubectl create secret generic -n osm-system egress-middle-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./middle.crt \
  --from-file=tls.key=./middle.key 
```

#### Setup Egress Policy

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: egress-middle
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-middle-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### Test Commands

Traffic flow:

Client --**tls**--> Nginx Ingress --**mtls**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -ksi https://ingress-nginx-controller.ingress-nginx/time --key /certs/client.key --cert /certs/client.crt
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/2 200 
date: Sun, 09 Oct 2022 08:41:21 GMT
content-type: text/plain; charset=utf-8
content-length: 75
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc
strict-transport-security: max-age=15724800; includeSubDomains

The current time: 2022-10-09 08:41:21.121578846 +0000 UTC m=+305.588353001
```

This business scenario is tested and the strategy is cleaned up to avoid affecting subsequent tests

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":null}}}' --type=merge

kubectl delete ingress -n egress-middle egress-middle
kubectl delete ingressbackend -n egress-middle egress-middle
kubectl delete egress -n egress-middle server-8443
kubectl delete secrets -n osm-system egress-middle-cert
kubectl delete secrets -n egress-middle nginx-cert-secret
```

### Scenario#4：mTLS Nginx & mTLS Ingress & mTLS Egress

#### Test Commands

Traffic flow:

Client --**http**--> Nginx Ingress Controller

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/1.1 404 Not Found
Date: Sun, 09 Oct 2022 08:41:52 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

#### Setup Ingress Controller Cert

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":{"secret":{"name":"ingress-controller-cert","namespace":"osm-system"},"subjectAltNames":["ingress-nginx.ingress-nginx.cluster.local"],"validityDuration":"24h"}}}}' --type=merge
```

#### Create Nginx CA Secret

```bash
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/ca.crt -o ca.crt

kubectl create secret generic -n egress-middle nginx-ca-secret \
  --from-file=ca.crt=./ca.crt 
```

#### Create Nginx TLS Secret

```bash
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/nginx.crt -o nginx.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/nginx.key -o nginx.key

kubectl create secret tls -n egress-middle nginx-cert-secret \
  --cert=./nginx.crt \
  --key=./nginx.key 
```

#### Setup Ingress Rules

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: egress-middle
  namespace: egress-middle
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "false"
    nginx.ingress.kubernetes.io/auth-tls-secret: egress-middle/nginx-ca-secret
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "middle.egress-middle.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/ingress-controller-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: middle
            port:
              number: 8080
  tls:
  - hosts:
    - ingress-nginx-controller.ingress-nginx
    secretName: nginx-cert-secret
EOF
```

#### Setup IngressBackend Policy

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of middle service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    namespace: ingress-nginx
    name: ingress-nginx-controller
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```

#### Test Commands

Traffic flow:

Client --**mtls**--> Nginx Ingress --**mtls** --> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -ksi https://ingress-nginx-controller.ingress-nginx/hello  --cacert /certs/ca.crt --key /certs/client.key --cert /certs/client.crt
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/2 200 
date: Sun, 09 Oct 2022 08:42:37 GMT
content-type: text/plain; charset=utf-8
content-length: 13
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc
strict-transport-security: max-age=15724800; includeSubDomains

hello world.
```

#### Disable Egress Permissive mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### Enable Egress Policy

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### Create Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/middle.crt -o middle.crt
curl https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/bidir-mtls/certs/middle.key -o middle.key

kubectl create secret generic -n osm-system egress-middle-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./middle.crt \
  --from-file=tls.key=./middle.key 
```

#### Setup Egress Policy

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: egress-middle
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-middle-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### Test Commands

Traffic flow:

Client --**mtls**--> Nginx Ingress --**mtls**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -ksi https://ingress-nginx-controller.ingress-nginx/time --cacert /certs/ca.crt --key /certs/client.key --cert /certs/client.crt
```

#### Test Results

The correct return result is similar to :

```bash
HTTP/2 200 
date: Sun, 09 Oct 2022 08:43:13 GMT
content-type: text/plain; charset=utf-8
content-length: 75
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc
strict-transport-security: max-age=15724800; includeSubDomains

The current time: 2022-10-09 08:43:13.241006879 +0000 UTC m=+417.741653620
```
