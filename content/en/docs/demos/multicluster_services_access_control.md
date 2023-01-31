---
title: "Multi-cluster services access control"
description: "Multi-cluster service access control"
type: docs
weight: 23
---

### Pre-requisites

- Tools and clusters created in demo [Multi-cluster services discovery & communication](/docs/demos/multicluster_services_communication)

This guide will expand on the knowledge we covered in previous guides and demonstrate how to configure and enable cross-cluster access control based on SMI. With osm-edge support for multi-clusters, users can define and enforce fine-grained access policies for services running across multiple Kubernetes clusters. This allows users to easily and securely manage access to services and resources, ensuring that only authorized users and applications have access to the appropriate services and resources.


Before we start, let's review the [SMI Access Control Specification](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md). There are two forms of [traffic policies](/docs/getting_started/traffic_policies/): **Permissive Mode** and **Traffic Policy Mode**. The former allows services in the mesh to access each other, while the latter requires the provision of the appropriate traffic policy to be accessible.

## SMI Access Control Policy

In traffic policy mode, SMI defines `ServiceAccount`-based access control through the Kubernetes Custom Resource Definition(CRD) `TrafficTarget`, which defines traffic sources (`sources`), destinations (`destinations`), and rules (`rules`). What is expressed is that applications that use the `ServiceAccount` specified in `sources` can access applications that have the `ServiceAccount` specified in `destinations`, and the accessible traffic is specified by `rules`.

For example, the following example represents a load running with `ServiceAccount` `promethues` sending a `GET` request to the `/metrics` endpoint of a load running with `ServiceAccount` `service-a`. The `HTTPRouteGroup` defines the identity of the traffic: i.e. the `GET` request to access the endpoint `/metrics`.

```yaml
kind: HTTPRouteGroup
metadata:
  name: the-routes
spec:
  matches:
  - name: metrics
    pathRegex: "/metrics"
    methods:
    - GET
---
kind: TrafficTarget
metadata:
  name: path-specific
  namespace: default
spec:
  destination:
    kind: ServiceAccount
    name: service-a
    namespace: default
  rules:
  - kind: HTTPRouteGroup
    name: the-routes
    matches:
    - metrics
  sources:
  - kind: ServiceAccount
    name: prometheus
    namespace: default
```

So how does access control perform in a multi-cluster?

### FSM's `ServiceExport`

FSM's `ServiceExport` is used to export services to other clusters, which is the process of service registration. The field `spec.serviceAccountName` of `ServiceExport` can be used to specify the `ServiceAccount` used for the service load.

```yml
apiVersion: flomesh.io/v1alpha1
kind: ServiceExport
metadata:
  namespace: httpbin
  name: httpbin
spec:
  serviceAccountName: "*"
  rules:
    - portNumber: 8080
      path: "/cluster-1/httpbin-mesh"
      pathType: Prefix
```

## Deploy the application

![mcs-access-control](/docs/images/mcs/16700408645693.png)

### Deploy the sample application

Deploy the `httpbin` application under the `httpbin` namespace (managed by the mesh, which injects sidecar) in clusters `cluster-1` and `cluster-3`. Here we specify `ServiceAccount` as `httpbin`.

```sh
export NAMESPACE=httpbin
for CLUSTER_NAME in cluster-1 cluster-3
do
  kubectx k3d-${CLUSTER_NAME}
  kubectl create namespace ${NAMESPACE}
  osm namespace add ${NAMESPACE}
  kubectl apply -n ${NAMESPACE} -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin
---  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  labels:
    app: pipy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy
  template:
    metadata:
      labels:
        app: pipy
    spec:
      serviceAccountName: httpbin
      containers:
        - name: pipy
          image: flomesh/pipy:latest
          ports:
            - containerPort: 8080
          command:
            - pipy
            - -e
            - |
              pipy()
              .listen(8080)
              .serveHTTP(new Message('Hi, I am from ${CLUSTER_NAME} and controlled by mesh!\n'))
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin-${CLUSTER_NAME}
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy
EOF

  sleep 3
  kubectl wait --for=condition=ready pod -n ${NAMESPACE} --all --timeout=60s
done
```

Deploy the `httpbin` application under the `cluster-2` namespace `httpbin`, but do not specify a `ServiceAccount` and use the default `ServiceAccount` `default`.

```sh
export NAMESPACE=httpbin
export CLUSTER_NAME=cluster-2

kubectx k3d-${CLUSTER_NAME}
kubectl create namespace ${NAMESPACE}
osm namespace add ${NAMESPACE}
kubectl apply -n ${NAMESPACE} -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  labels:
    app: pipy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy
  template:
    metadata:
      labels:
        app: pipy
    spec:
      containers:
        - name: pipy
          image: flomesh/pipy:latest
          ports:
            - containerPort: 8080
          command:
            - pipy
            - -e
            - |
              pipy()
              .listen(8080)
              .serveHTTP(new Message('Hi, I am from ${CLUSTER_NAME}! and controlled by mesh!\n'))
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin-${CLUSTER_NAME}
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy
EOF

sleep 3
kubectl wait --for=condition=ready pod -n ${NAMESPACE} --all --timeout=60s
```

Deploy the `curl` application under the namespace `curl` in cluster `cluster-2`, which is managed by the mesh, and the injected sidecar will be fully traffic dispatched across the cluster. Specify here to use `ServiceAccout` `curl`.

```sh
export NAMESPACE=curl
kubectx k3d-cluster-2
kubectl create namespace ${NAMESPACE}
osm namespace add ${NAMESPACE}
kubectl apply -n ${NAMESPACE} -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: curl
---
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
---
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
      containers:
      - image: curlimages/curl
        imagePullPolicy: IfNotPresent
        name: curl
        command: ["sleep", "365d"]
EOF

sleep 3
kubectl wait --for=condition=ready pod -n ${NAMESPACE} --all --timeout=60s
```

### Export service

```sh
export NAMESPACE_MESH=httpbin
for CLUSTER_NAME in cluster-1 cluster-3
do
  kubectx k3d-${CLUSTER_NAME}
  kubectl apply -f - <<EOF
apiVersion: flomesh.io/v1alpha1
kind: ServiceExport
metadata:
  namespace: ${NAMESPACE_MESH}
  name: httpbin
spec:
  serviceAccountName: "httpbin"
  rules:
    - portNumber: 8080
      path: "/${CLUSTER_NAME}/httpbin-mesh"
      pathType: Prefix
---
apiVersion: flomesh.io/v1alpha1
kind: ServiceExport
metadata:
  namespace: ${NAMESPACE_MESH}
  name: httpbin-${CLUSTER_NAME}
spec:
  serviceAccountName: "httpbin"
  rules:
    - portNumber: 8080
      path: "/${CLUSTER_NAME}/httpbin-mesh-${CLUSTER_NAME}"
      pathType: Prefix
EOF
sleep 1
done
```

### Test

We switch back to the cluster `cluster-2`.

```sh
kubectx k3d-cluster-2
```

The default route type is `Locality` and we need to create an `ActiveActive` policy to allow requests to be processed using service instances from other clusters.

```sh
kubectl apply -n httpbin -f  - <<EOF
apiVersion: flomesh.io/v1alpha1
kind: GlobalTrafficPolicy
metadata:
  name: httpbin
spec:
  lbType: ActiveActive
  targets:
    - clusterKey: default/default/default/cluster-1
    - clusterKey: default/default/default/cluster-3
EOF
```

In the `curl` application of the `cluster-2` cluster, we send a request to `httpbin.httpbin`.

```sh
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/
```

A few more requests will see the following response.

```sh
Hi, I am from cluster-1 and controlled by mesh!
Hi, I am from cluster-2 and controlled by mesh!
Hi, I am from cluster-3 and controlled by mesh!
Hi, I am from cluster-1 and controlled by mesh!
Hi, I am from cluster-2 and controlled by mesh!
Hi, I am from cluster-3 and controlled by mesh!
```

## Demo

### Adjusting the traffic policy mode

Let's adjust the traffic policy mode of cluster `cluster-2` so that the traffic policy can be applied.

```sh
kubectx k3d-cluster-2
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}' --type=merge
```

In this case, if you try to send the request again, you will find that the request fails. This is because in traffic policy mode, inter-application access is prohibited if no policy is configured.

```sh
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/
command terminated with exit code 52
```

### Application Access Control Policy

The access control policy of SMI is based on `ServiceAccount` as mentioned at the beginning of the guide, that's why our deployed `httpbin` service uses different `ServiceAccount` in cluster `cluster-1`, `cluster-3`, and cluster `cluster-2`.

- cluster-1：`httpbin`
- cluster-2：`default`
- cluster-3：`httpbin`

Next, we will set different access control policies `TrafficTarget` for in-cluster and out-of-cluster services, differentiated by the `ServiceAccount` of the target load in `TrafficTarget`.

![fsm-multi-cluster-with-policy](16700408453085/16700408645711.png)

Execute the following command to create a traffic policy `curl-to-httpbin` that allows `curl` to access loads under the namespace `httpbin` that uses `ServiceAccount` `default`.

```sh
kubectl apply -n httpbin -f - <<EOF
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: httpbin-route
spec:
  matches:
  - name: all
    pathRegex: "/"
    methods:
    - GET
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: curl-to-httpbin
spec:
  destination:
    kind: ServiceAccount
    name: default
    namespace: httpbin
  rules:
  - kind: HTTPRouteGroup
    name: httpbin-route
    matches:
    - all
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: curl
EOF
```

Multiple request attempts are sent and the service of cluster `cluster-2` responds, while clusters `cluster-1` and `cluster-3` will not participate in the service.

```shell
Hi, I am from cluster-2 and controlled by mesh!
Hi, I am from cluster-2 and controlled by mesh!
Hi, I am from cluster-2 and controlled by mesh!
```

Execute the following command to check `ServiceImports` and you can see that `cluster-1` and `cluster-3` export services using `ServiceAccount` `httpbin`.

```sh
kubectl get serviceimports httpbin -n httpbin -o jsonpath='{.spec}' | jq

{
  "ports": [
    {
      "endpoints": [
        {
          "clusterKey": "default/default/default/cluster-1",
          "target": {
            "host": "192.168.1.110",
            "ip": "192.168.1.110",
            "path": "/cluster-1/httpbin-mesh",
            "port": 81
          }
        },
        {
          "clusterKey": "default/default/default/cluster-3",
          "target": {
            "host": "192.168.1.110",
            "ip": "192.168.1.110",
            "path": "/cluster-3/httpbin-mesh",
            "port": 83
          }
        }
      ],
      "port": 8080,
      "protocol": "TCP"
    }
  ],
  "serviceAccountName": "httpbin",
  "type": "ClusterSetIP"
}
```

So, we create another `TrafficTarget` `curl-to-ext-httpbin` that allows `curl` to access the load using `ServiceAccount` `httpbin`.

```sh
kubectl apply -n httpbin -f - <<EOF
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: curl-to-ext-httpbin
spec:
  destination:
    kind: ServiceAccount
    name: httpbin
    namespace: httpbin
  rules:
  - kind: HTTPRouteGroup
    name: httpbin-route
    matches:
    - all
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: curl
EOF
```

After applying the policy, test it again and all requests are successful.

```sh
Hi, I am from cluster-2 and controlled by mesh!
Hi, I am from cluster-1 and controlled by mesh!
Hi, I am from cluster-3 and controlled by mesh!
```

