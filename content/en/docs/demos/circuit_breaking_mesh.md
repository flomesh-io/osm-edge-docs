---
title: "Circuit breaking for destinations within the mesh"
description: "Configuring circuit breaking for destinations within the mesh"
weight: 21
---

This guide demonstrates how to configure circuit breaking for destinations that are a part of an osm-edge managed service mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have osm-edge installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- osm-edge version >= v1.3.0.


## Demo

The following demo shows a load-testing client [fortio](https://github.com/fortio/fortio) sending traffic to the `httpbin` service. We will see how applying circuit breakers for traffic to the `httpbin` service impacts the `fortio` client when the configured circuit breaking limits trip.

For simplicity, enable [permissive traffic policy mode](/docs/guides/traffic_management/permissive_mode) so that explicit SMI traffic access policies are not required for application connectivity within the mesh.

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
```

### Deploy services

```shell
kubectl create namespace server
osm namespace add server
kubectl apply -n server -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: fortio
  labels:
    app: fortio
    service: fortio
spec:
  ports:
  - port: 8080
    name: http-8080
  - port: 8078
    name: tcp-8078
  - port: 8079
    name: grpc-8079
  selector:
    app: fortio
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio
  template:
    metadata:
      labels:
        app: fortio
    spec:
      containers:
      - name: fortio
        image: fortio/fortio:latest_release
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8078
          name: tcp
        - containerPort: 8079
          name: grpc
EOF
```

Deploy client service.

```shell
kubectl create namespace client
osm namespace add client

kubectl apply -n client -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fortio-client
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio-client
  template:
    metadata:
      labels:
        app: fortio-client
    spec:
      serviceAccountName: fortio-client
      containers:
      - name: fortio-client
        image: fortio/fortio:latest_release
        imagePullPolicy: Always
EOF
```

### Test

Confirm the `fortio-client` is able to successfully make HTTP requests to the `fortio-server` service on port `8080`. We call the service with `10` concurrent connections (`-c 10`) and send `1000` requests (`-n 1000`). Service is configured to return HTTP status code of `511` for `20%` requests.

```shell
fortio_client=`kubectl get pod -n client -l app=fortio-client -o jsonpath='{.items[0].metadata.name}'`
kubectl exec "$fortio_client" -n client -c fortio-client -- fortio load -quiet -c 10 -n 1000 -qps 200 -p 99.99 http://fortio.server.svc.cluster.local:8080/echo?status=511:20
```

Returned result might look something like below:

```
Sockets used: 205 (for perfect keepalive, would be 10)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.43.43.151:8080: 205
Code 200 : 804 (80.4 %)
Code 511 : 196 (19.6 %)
All done 1000 calls (plus 0 warmup) 2.845 ms avg, 199.8 qps
```

Next, apply a circuit breaker configuration using the `UpstreamTrafficSetting` resource for traffic directed to the `fortio-server` service.

### Policy: Error Request Count Triggers Circuit Breaker

The error request count trigger threshold is set to `errorAmountThreshold=100`, the circuit breaker is triggered when the error request count reaches `100`, returning `503 Service Unavailable!`, the circuit breaker lasts for `10s`

```shell
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-circuit-breaking
  namespace: server
spec:
  host: fortio.server.svc.cluster.local
  connectionSettings:
    http:
      circuitBreaking:
        statTimeWindow: 1m
        minRequestAmount: 200
        errorAmountThreshold: 100
        degradedTimeWindow: 10s
        degradedStatusCode: 503
        degradedResponseContent: 'Service Unavailable!'
EOF
```

Set `20%` of the server's responses to return error code `511`. When the error request count reaches `100`, the number of successful requests should be around `400`.

```shell
kubectl exec "$fortio_client" -n client -c fortio-client -- fortio load -quiet -c 10 -n 1000 -qps 200 -p 99.99 http://fortio.server.svc.cluster.local:8080/echo\?status\=511:20
```

From the results, it meets the expectations. After circuit breaker, the requests return the `503` error code.

```shell
Sockets used: 570 (for perfect keepalive, would be 10)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.43.43.151:8080: 570
Code 200 : 430 (43.0 %)
Code 503 : 470 (47.0 %)
Code 511 : 100 (10.0 %)
All done 1000 calls (plus 0 warmup) 3.376 ms avg, 199.8 qps
```

Checking the sidecar logs, the error count reached 100 at the 530th request, triggering the circuit breaker.

```shell
2023-02-08 01:08:01.456 [INF] [circuit_breaker] total/slowAmount/errorAmount (open) server/fortio|8080 530 0 100
```

### Policy: Error Rate Triggers Circuit Breaker

Here we change the trigger from error count to error rate: `errorRatioThreshold=0.10`, the circuit breaker is triggered when the error rate reaches `10%`, it lasts for `10s` and returns `503 Service Unavailable!`. Note that the minimum number of requests is still `200`.

```shell
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-circuit-breaking
  namespace: server
spec:
  host: fortio.server.svc.cluster.local
  connectionSettings:
    http:
      circuitBreaking:
        statTimeWindow: 1m
        minRequestAmount: 200
        errorRatioThreshold: 0.10
        degradedTimeWindow: 10s
        degradedStatusCode: 503
        degradedResponseContent: 'Service Unavailable!'
EOF
```

Set `20%` of the server's responses to return error code `511`.

````shell
kubectl exec "$fortio_client" -n client -c fortio-client -- fortio load -quiet -c 10 -n 1000 -qps 200 -p 99.99 http://fortio.server.svc.cluster.local:8080/echo\?status\=511:20
````

From the output results, it can be seen that after 200 requests, the circuit breaker was triggered, and the number of circuit breaker requests was 800.

```shell
Sockets used: 836 (for perfect keepalive, would be 10)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.43.43.151:8080: 836
Code 200 : 164 (16.4 %)
Code 503 : 800 (80.0 %)
Code 511 : 36 (3.6 %)
All done 1000 calls (plus 0 warmup) 3.605 ms avg, 199.8 qps
```

Checking the sidecar logs, after satisfying the minimum number of requests to trigger the circuit breaker (200), the error rate also reached the threshold, triggering the circuit breaker.

```
2023-02-08 01:19:25.874 [INF] [circuit_breaker] total/slowAmount/errorAmount (close) server/fortio|8080 200 0 36
```

### Policy: Slow Call Request Count Triggers Circuit Breaker

Testing slow calls, add a `200ms` delay to `20%` of the requests.

```shell
kubectl exec "$fortio_client" -n client -c fortio-client -- fortio load -quiet -c 10 -n 1000 -qps 200 -p 50,78,79,80,81,82,90,95 http://fortio.server.svc.cluster.local:8080/echo\?delay\=200ms:20
```

A similar effect is achieved, with nearly 80% of requests taking less than 200ms.

```shell
# target 50% 0.000999031
# target 78% 0.0095
# target 79% 0.200175
# target 80% 0.200467
# target 81% 0.200759
# target 82% 0.20105
# target 90% 0.203385
# target 95% 0.204844

285103 max 0.000285103 sum 0.000285103
Sockets used: 10 (for perfect keepalive, would be 10)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.43.43.151:8080: 10
Code 200 : 1000 (100.0 %)
All done 1000 calls (plus 0 warmup) 44.405 ms avg, 149.6 qps
```

Setting strategy:

- Set the slow call request time threshold to `200ms`
- Set the number of slow call requests to `100`

```shell
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-circuit-breaking
  namespace: server
spec:
  host: fortio.server.svc.cluster.local
  connectionSettings:
    http:
      circuitBreaking:
        statTimeWindow: 1m
        minRequestAmount: 200
        slowTimeThreshold: 200ms
        slowAmountThreshold: 100
        degradedTimeWindow: 10s
        degradedStatusCode: 503
        degradedResponseContent: 'Service Unavailable!'
EOF
```

Inject a 200ms delay for 20% of requests.

```shell
kubectl exec "$fortio_client" -n client -c fortio-client -- fortio load -quiet -c 10 -n 1000 -qps 200 -p 50,78,79,80,81,82,90,95 http://fortio.server.svc.cluster.local:8080/echo\?delay\=200ms:20
```

The number of successful requests is 504, with 20% taking 200ms, and the number of slow requests has reached the threshold to trigger a circuit breaker.

```shell
# target 50% 0.00246111
# target 78% 0.00393846
# target 79% 0.00398974
# target 80% 0.00409756
# target 81% 0.00421951
# target 82% 0.00434146
# target 90% 0.202764
# target 95% 0.220036

Sockets used: 496 (for perfect keepalive, would be 10)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.43.43.151:8080: 496
Code 200 : 504 (50.4 %)
Code 503 : 496 (49.6 %)
All done 1000 calls (plus 0 warmup) 24.086 ms avg, 199.8 qps
```

Checking the logs of the sidecar, the number of slow calls reached 100 on the 504th request, triggering the circuit breaker.

```shell
2023-02-08 07:27:01.106 [INF] [circuit_breaker] total/slowAmount/errorAmount (open)  server/fortio|8080 504 100 0
```

### Policy: Slow Call Rate Triggers Circuit Breaking

The slow call rate is set to `0.10`, meaning that the circuit breaker will be triggered when it is expected that `10%` of the requests within the statistical window will take more than `200ms`.

```shell
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-circuit-breaking
  namespace: server
spec:
  host: fortio.server.svc.cluster.local
  connectionSettings:
    http:
      circuitBreaking:
        statTimeWindow: 1m
        minRequestAmount: 200
        slowTimeThreshold: 200ms
        slowRatioThreshold: 0.1
        degradedTimeWindow: 10s
        degradedStatusCode: 503
        degradedResponseContent: 'Service Unavailable!'
EOF
```

Add 200ms delay to 20% of requests.

```shell
kubectl exec "$fortio_client" -n client -c fortio-client -- fortio load -quiet -c 10 -n 1000 -qps 200 -p 50,78,79,80,81,82,90,95 http://fortio.server.svc.cluster.local:8080/echo\?delay\=200ms:20
```

From the output results, there are 202 successful requests which meets the minimum number of requests required for the configuration to take effect. Among them, 20% are slow calls, reaching the threshold for triggering circuit breaker.

```shell
# target 50% 0.00305539
# target 78% 0.00387172
# target 79% 0.00390087
# target 80% 0.00393003
# target 81% 0.00395918
# target 82% 0.00398834
# target 90% 0.00458915
# target 95% 0.00497674

Sockets used: 798 (for perfect keepalive, would be 10)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.43.43.151:8080: 798
Code 200 : 202 (20.2 %)
Code 503 : 798 (79.8 %)
All done 1000 calls (plus 0 warmup) 10.133 ms avg, 199.8 qps
```

Check the logs of the sidecar, at the 202nd request, the number of slow requests was 28, which triggered the circuit breaker.

```shell
2023-02-08 07:38:25.284 [INF] [circuit_breaker] total/slowAmount/errorAmount (open)  server/fortio|8080 202 28 0
```




