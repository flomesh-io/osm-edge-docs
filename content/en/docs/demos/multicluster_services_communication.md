---
title: "Multi-cluster services discovery & communication"
description: "Multi-cluster service communication using Flomesh Service Mesh"
type: docs
weight: 23
---

## Demo Architecture

For demonstration purposes we will be creating 4 Kubernetes clusters and high-level architecture will look something like the below:

![](/docs/images/mcs/16693549955635.png)

> As a convention and for this demo we will be creating a separate stand-alone cluster to serve as a **control plane** cluster, but that isn't strictly required as a separate cluster and it could be one of any existing cluster.


### Pre-requisites

- `kubectx`: for switching between multiple `kubeconfig contexts` (clusters)
- `k3d`: for creating multiple `k3s` clusters locally using containers
- `helm`: for deploying `FSM`
- `docker`: required to run `k3d`
- Have `osm` CLI available for managing the service mesh.
- OSM version >= v1.2.0.

### Demo clusters & environment setup

In this demo, we will be using [k3d](https://k3d.io/) a lightweight wrapper to run [k3s](https://github.com/rancher/k3s) (Rancher Lab’s minimal Kubernetes distribution) in docker, to create 4 separate clusters named `control-plane`, `cluster-1`, `cluster-2`, and `cluster-3` respectively.

We will be using the HOST machine IP address and separate ports during the installation, for us to easily access the individual clusters. My demo host machine IP address is `192.168.1.110` (it might be different for your machine).


| cluster | cluster ip | api-server port | LB external-port | description |
|---------------|:-----------------------|:--------------------|:------------|:-------------|
| control-plane | HOST_IP(192.168.1.110) | 6444 | N/A | control-plane cluster |
| cluster-1 | HOST_IP(192.168.1.110) | 6445 | 81 | application-cluster |
| cluster-2 | HOST_IP(192.168.1.110) | 6446 | 82 | Application Cluster |
| cluster-3 | HOST_IP(192.168.1.110) | 6447 | 83 | Application Cluster |

### Network

Creates a docker `bridge` type network named `multi-clusters`, which run all containers.

```sh
docker network create multi-clusters
```

Find your machine host IP address, mine is `192.168.1.110`, and export that as an environment variable to be used later.

```sh
export HOST_IP=192.168.1.110
```

### Cluster creation

We are going to use `k3d` to create 4 clusters.

```sh
API_PORT=6444 #6444 6445 6446 6447
PORT=80 #81 82 83
for CLUSTER_NAME in control-plane cluster-1 cluster-2 cluster-3
do
  k3d cluster create ${CLUSTER_NAME} \
    --image docker.io/rancher/k3s:v1.23.8-k3s2 \
    --api-port "${HOST_IP}:${API_PORT}" \
    --port "${API_PORT}:6443@server:0" \
    --port "${PORT}:80@server:0" \
    --servers-memory 4g \
    --k3s-arg "--disable=traefik@server:0" \
    --network multi-clusters \
    --timeout 120s \
    --wait
    ((API_PORT=API_PORT+1))
    ((PORT=PORT+1))
done
```

### Install FSM

Install FSM to newly created 4 clusters.

```sh
helm repo update
export FSM_NAMESPACE=flomesh
export FSM_VERSION=0.2.0-alpha.9
for CLUSTER_NAME in control-plane cluster-1 cluster-2 cluster-3
do 
  kubectx k3d-${CLUSTER_NAME}
  sleep 1
  helm install --namespace ${FSM_NAMESPACE} --create-namespace --version=${FSM_VERSION} --set fsm.logLevel=5 fsm fsm/fsm
  sleep 1
  kubectl wait --for=condition=ready pod --all -n $FSM_NAMESPACE
done
```

We have our clusters ready, now we need to federate them together, but before we do that, let's first understand the mechanics on how FSM is configured.

### Federate clusters

We will enroll clusters `cluster-1`, `cluster-2`, and `cluster-3` into the management of  `control-plane` cluster.


```sh
export HOST_IP=192.168.1.110
kubectx k3d-control-plane
sleep 1
PORT=81
for CLUSTER_NAME in cluster-1 cluster-2 cluster-3
do
  cat <<EOF
apiVersion: flomesh.io/v1alpha1
kind: Cluster
metadata:
  name: ${CLUSTER_NAME}
spec:
  gatewayHost: ${HOST_IP}
  gatewayPort: ${PORT}
  kubeconfig: |+
`k3d kubeconfig get ${CLUSTER_NAME} | sed 's|^|    |g' | sed "s|0.0.0.0|$HOST_IP|g"`
EOF
((PORT=PORT+1))
done
```

### Install osm-edge Service Mesh

Download osm CLI

Install the service mesh osm-edge to the clusters `cluster-1`, `cluster-2`, and `cluster-3`. The control plane does not handle application traffic and does not need to be installed.

```sh
export OSM_NAMESPACE=osm-system
export OSM_MESH_NAME=osm
for CLUSTER_NAME in cluster-1 cluster-2 cluster-3
do
  kubectx k3d-${CLUSTER_NAME}
  DNS_SVC_IP="$(kubectl get svc -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[0].spec.clusterIP}')"
osm install \
    --mesh-name "$OSM_MESH_NAME" \
    --osm-namespace "$OSM_NAMESPACE" \
    --set=osm.certificateProvider.kind=tresor \
    --set=osm.image.pullPolicy=Always \
    --set=osm.sidecarLogLevel=error \
    --set=osm.controllerLogLevel=warn \
    --timeout=900s \
    --set=osm.localDNSProxy.enable=true \
    --set=osm.localDNSProxy.primaryUpstreamDNSServerIPAddr="${DNS_SVC_IP}"
done
```

## Deploy Demo application

### Deploying mesh-managed applications

Deploy the `httpbin` application under the `httpbin` namespace of clusters `cluster-1` and `cluster-3` (which are managed by the mesh and will inject sidecar). Here the `httpbin` application is implemented by [Pipy](https://flomesh.io/pipy) and will return the current cluster name.

```sh
export NAMESPACE=httpbin
for CLUSTER_NAME in cluster-1 cluster-3
do
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

Deploy the `curl` application under the namespace `curl` in cluster `cluster-2`, which is managed by the mesh.

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

### Export Service

Let's export services in `cluster-1` and `cluster-3`

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
  serviceAccountName: "*"
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
  serviceAccountName: "*"
  rules:
    - portNumber: 8080
      path: "/${CLUSTER_NAME}/httpbin-mesh-${CLUSTER_NAME}"
      pathType: Prefix
EOF
sleep 1
done
```

After exporting the services, FSM will automatically create Ingress rules for them, and with the rules, you can access these services through Ingress.

```sh

for CLUSTER_NAME_INDEX in 1 3
do
  CLUSTER_NAME=cluster-${CLUSTER_NAME_INDEX}
  ((PORT=80+CLUSTER_NAME_INDEX))
  kubectx k3d-${CLUSTER_NAME}
  echo "Getting service exported in cluster ${CLUSTER_NAME}"
  echo '-----------------------------------'
  kubectl get serviceexports.flomesh.io -A
  echo '-----------------------------------'
  curl -s "http://${HOST_IP}:${PORT}/${CLUSTER_NAME}/httpbin-mesh"
  curl -s "http://${HOST_IP}:${PORT}/${CLUSTER_NAME}/httpbin-mesh-${CLUSTER_NAME}"
  echo '-----------------------------------'
done
```

To view one of the `ServiceExports` resources.

```sh
kubectl get serviceexports httpbin -n httpbin -o jsonpath='{.spec}' | jq

{
  "loadBalancer": "RoundRobinLoadBalancer",
  "rules": [
    {
      "path": "/cluster-3/httpbin-mesh",
      "pathType": "Prefix",
      "portNumber": 8080
    }
  ],
  "serviceAccountName": "*"
}
```

The exported services can be imported into other managed clusters. For example, if we look at the cluster `cluster-2`, we can have multiple services imported.

```sh
kubectx k3d-cluster-2
kubectl get serviceimports -A

NAMESPACE   NAME                AGE
httpbin     httpbin-cluster-1   13m
httpbin     httpbin-cluster-3   13m
httpbin     httpbin             13m
```

### Testing

Staying in the `cluster-2` cluster (`kubectx k3d-cluster-2`), we test if we can access these imported services from the `curl` application in the mesh.

Get the pod of the `curl` application, from which we will later launch requests to simulate service access.

```sh
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
```

At this point you will find that it is not accessible.

```sh
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/

command terminated with exit code 7
```

Note that this is normal, by default no other cluster instance will be used to respond to requests, which means no calls to other clusters will be made by default. So how to access it, then we need to be clear about the global traffic policy `GlobalTrafficPolicy`.


## Global Traffic Policy

Note that all global traffic policies are set on the user's side, so this demo is about setting global traffic policies on the cluster `cluster-2` side. So before you start, switch to cluster `cluster-2`: `kubectx k3d-cluster-2`.

The global traffic policy is set via CRD `GlobalTrafficPolicy`.

```go
type GlobalTrafficPolicy struct {  
   metav1.TypeMeta   `json:",inline"`  
   metav1.ObjectMeta `json:"metadata,omitempty"`  
  
   Spec   GlobalTrafficPolicySpec   `json:"spec,omitempty"`  
   Status GlobalTrafficPolicyStatus `json:"status,omitempty"`  
}
type GlobalTrafficPolicySpec struct {  
   LbType LoadBalancerType `json:"lbType"`  
   LoadBalanceTarget []TrafficTarget `json:"targets"`  
}
```

Global load balancing types `.spec.lbType` There are three types.

- `Locality`: uses only the services of this cluster, and is also the default type. This is why accessing the `httpbin` application fails when we don't provide any global policy, because there is no such service in cluster `cluster-2`.
- `FailOver`: proxies to other clusters only when access to this cluster fails, which is often referred to as failover, similar to primary backup.
- `ActiveActive`: Proxy to other clusters under normal conditions, similar to multi-live.

The `FailOver` and `ActiveActive` policies are used with the *targets* field to specify the id of the standby cluster, which is the cluster that can be routed to in case of failure or load balancing. ** For example, if you look at the import service `httpbin/httpbin` in cluster `cluster-2`, you can see that it has two `endpoints` for the outer cluster, note that `endpoints` here is a different concept than the native `endpoints.v1` and will contain more information. In addition, there is the cluster id `clusterKey`.

```shell
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
  "serviceAccountName": "*",
  "type": "ClusterSetIP"
}
```

### Routing Type - `Locality`

The default routing type is `Locality`, and as tested above, traffic cannot be dispatched to other clusters.

```shell
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/
command terminated with exit code 7
```

### Routing Type - `FailOver`

Since setting a global traffic policy for causes access failure, we start by enabling `FailOver` mode. **Note that the global policy traffic, to be consistent with the target service name and namespace.** For example, if we want to access `http://httpbin.httpbin:8080/`, we need to create `GlobalTrafficPolicy` resource named `httpbin` under the namespace `httpbin`.

```shell
kubectl apply -n httpbin -f  - <<EOF
apiVersion: flomesh.io/v1alpha1
kind: GlobalTrafficPolicy
metadata:
  name: httpbin
spec:
  lbType: FailOver
  targets:
    - clusterKey: default/default/default/cluster-1
    - clusterKey: default/default/default/cluster-3
EOF
```

After setting the policy, let's try it again by requesting.

```shell
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/
Hi, I am from cluster-1!
```

The request is successful and the request is proxied to the service in cluster `cluster-1`. Another request is made, and it is proxied to cluster `cluster-3`, as expected for load balancing.

```shell
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/
Hi, I am from cluster-3!
```

What will happen if we deploy the application `httpbin` in the namespace `httpbin` of the cluster `cluster-2`?

```shell
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
              .serveHTTP(new Message('Hi, I am from ${CLUSTER_NAME}!\n'))
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

After the application is running normally, this time we send the request to test again. From the results, it looks like the request is processed in the current cluster.

```shell
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/ 
Hi, I am from cluster-2!
```

Even if the request is repeated multiple times, it will always return `Hi, I am from cluster-2!`, which indicates that the services of same cluster are used in preference to the services imported from other clusters.

In some cases, we also want other clusters to participate in the service as well, because the resources of other clusters are wasted if only the services of this cluster are used. This is where the `ActiveActive` routing type comes into play.

### Routing Type - `ActiveActive`

Moving on from the status above, let's test the `ActiveActive` type by modifying the policy created earlier and updating it to `ActiveActive`: `ActiveActive

```shell
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

Multiple requests will show that `httpbin` from all three clusters will participate in the service. This indicates that the load is being proxied to multiple clusters in a balanced manner.

```sh
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/
Hi, I am from cluster-1 and controlled by mesh!
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/
Hi, I am from cluster-2!
kubectl exec "${curl_client}" -n curl -c curl -- curl -s http://httpbin.httpbin:8080/
Hi, I am from cluster-3 and controlled by mesh!
```