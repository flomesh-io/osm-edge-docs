---
title: "eBPF setup quickstart"
description: "Learn how to use eBPF with osm-edge"
type: docs
weight: 2
---

# Using eBPF with osm-edge

This quick start guide demonstrates how to setup environment and configure osm-edge to use eBPF as its interception method.

For more details refer to [Learn how to use eBPF with osm-edge](/docs/getting_started/ebpf)

## Prerequisites

- Ubuntu 20.04
- Kernel 5.15.0-1034
- 2c4g VM * 3：master、node1、node2

### Install CNI Plugin

Execute the following command on all nodes to download the CNI plugin.

```shell
sudo mkdir -p /opt/cni/bin
curl -sSL https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz | sudo tar -zxf - -C /opt/cni/bin
```

### Master Node

Get the IP address of the master node. (Your machine IP might be different)

```shell
export MASTER_IP=10.0.2.6
```

Kubernetes cluster uses the k3s distribution, but when installing the cluster, you need to disable the flannel integrated by k3s and use independently installed flannel for validation. This is because k3s's doesn't follow Flannel directory structure `/opt/cni/bin` and store its CNI bin directory at `/var/lib/rancher/k3s/data/xxx/bin` where `xxx` is some randomly generated text.

```shell
curl -sfL https://get.k3s.io | sh -s - --disable traefik --disable servicelb --flannel-backend=none --advertise-address $MASTER_IP --write-kubeconfig-mode 644 --write-kubeconfig ~/.kube/config
```

Install Flannel. **Note that the default Pod CIDR of Flannel is `10.244.0.0/16`, and we will modify it to k3s's default `10.42.0.0/16`.**

```shell
curl -s https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml | sed 's|10.244.0.0/16|10.42.0.0/16|g' | kubectl apply -f -
```

Get the access token of the API server for initializing worker nodes.

```shell
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Worker Node
Use the IP address of the master node and the token obtained earlier to initialize the node.

```shell
export INSTALL_K3S_VERSION=v1.23.8+k3s2
export NODE_TOKEN=K107c1890ae060d191d347504740566f9c506b95ea908ba4795a7a82ea2c816e5dc::server:2757787ec4f9975ab46b5beadda446b7
curl -sfL https://get.k3s.io | K3S_URL=https://${MASTER_IP}:6443 K3S_TOKEN=${NODE_TOKEN} sh -
```

### Download `osm-edge` CLI

```shell
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.3.3
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
sudo cp ./${system}-${arch}/osm /usr/local/bin/
```

### Install osm-edge

```bash
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.trafficInterceptionMode=ebpf \
    --set=osm.osmInterceptor.debug=true \
    --timeout=900s
```

### Deploy Sample Application

```bash
#Sample services
kubectl create namespace ebpf
osm namespace add ebpf
kubectl apply -n ebpf -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/main/demo/interceptor/curl.yaml
kubectl apply -n ebpf -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/main/demo/interceptor/pipy-ok.yaml

#Schedule Pods to Different Nodes
kubectl patch deployments curl -n ebpf -p '{"spec":{"template":{"spec":{"nodeName":"node1"}}}}'
kubectl patch deployments pipy-ok-v1 -n ebpf -p '{"spec":{"template":{"spec":{"nodeName":"node1"}}}}'
kubectl patch deployments pipy-ok-v2 -n ebpf -p '{"spec":{"template":{"spec":{"nodeName":"node2"}}}}'

sleep 5

#Wait for dependent Pods to start successfully
kubectl wait --for=condition=ready pod -n ebpf -l app=curl --timeout=180s
kubectl wait --for=condition=ready pod -n ebpf -l app=pipy-ok -l version=v1 --timeout=180s
kubectl wait --for=condition=ready pod -n ebpf -l app=pipy-ok -l version=v2 --timeout=180s
```

### Testing

During testing, you can view the debug logs of BPF program execution by viewing the kernel tracing logs on the worker node using the following command. To avoid interference caused by sidecar communication with the control plane, first obtain the IP address of the control plane.

```shell
kubectl get svc -n osm-system osm-controller -o jsonpath='{.spec.clusterIP}'
10.43.241.189
```

Execute the following command on both worker nodes.

```shell
sudo cat /sys/kernel/debug/tracing/trace_pipe | grep bpf_trace_printk | grep -v '10.43.241.189'
```

Execute the following command on both worker nodes.

```shell
curl_client="$(kubectl get pod -n ebpf -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n ebpf -c curl -- curl -s pipy-ok:8080
```

You should receive results similar to the following, and the kernel tracing logs should also output the debug logs of the BPF program accordingly (the content is quite long, so it will not be shown here).

```shell
Hi, I am pipy ok v1 !
Hi, I am pipy ok v2 !
```