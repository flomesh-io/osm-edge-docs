---
title: "Using eBPF with osm-edge"
description: "Learn how to use eBPF with osm-edge"
weight: 2
---

# Using eBPF for traffic interception and communication

osm-edge introduced eBPF functinoality starting from its version [Flomesh Service Mesh 1.3.3](https://github.com/flomesh-io/osm-edge/releases/tag/v1.3.3) and now users can opt-in for eBPF or go with default iptables implementation.

This guide shows how to start using this new functionality and enjoy the benefits eBPF. Before we get our hands dirty with demo, let's have a quick review of pros and cons of each approach.

> If you want to directly jump into quick start, refer to [eBPF setup quickstart guide](/docs/quickstart/ebpf)

In service mesh, iptables and eBPF are two common ways of intercepting traffic.

iptables is a traffic interception tool based on the Linux kernel. It can control traffic by filtering rules. Its advantages include:

- Universality: The iptables tool has been widely used in Linux operating systems, so most Linux users are familiar with its usage.
- Stability: iptables has long been part of the Linux kernel, so it has a high degree of stability.
- Flexibility: iptables can be flexibly configured according to needs to control network traffic.


However, iptables also has some disadvantages:

- Difficult to debug: Due to the complexity of the iptables tool itself, it is relatively difficult to debug.
- Performance issues: iptables performs processing in the kernel space, which may affect network performance.
- Issues with handling complex traffic: When it comes to handling complex traffic, iptables may not be suitable because its rule processing is not flexible enough.

eBPF is an advanced traffic interception tool that can intercept and analyze traffic in the Linux kernel through custom programs. The advantages of eBPF include:

- Flexibility: eBPF can use custom programs to intercept and analyze traffic, so it has higher flexibility.
- Scalability: eBPF can dynamically load and unload programs, so it has higher scalability.
- Efficiency: eBPF can perform processing in the kernel space, so it has higher performance.

However, eBPF also has some disadvantages:

- Higher learning curve: eBPF is relatively new compared to iptables, so it requires some learning costs.
- Complexity: Developing custom eBPF programs may be more complex.
- Security: Since eBPF can directly manipulate the kernel, it needs to be used with caution to ensure security.

Overall, iptables is more suitable for simple traffic filtering and management, while eBPF is more suitable for complex traffic interception and analysis scenarios that require higher flexibility and performance.

## Architecture

To provide eBPF features in 1.3.3, Flomesh Service Mesh provides the osm-cni CNI implementation and osm-interceptor running on each node, where osm-cni is compatible with mainstream CNI plugins.

When kubelet creates a pod on a node, it calls the CNI interface through the container runtime CRI to create the pod's network namespace. After the pod's network namespace is created, osm-cni calls the interface of osm-interceptor to load the BPF program and attach it to the hook point. In addition, osm-interceptor also maintains pod information in eBPF Maps.

![](/docs/images/ebpf/architecture.png)

## Implementation Principles

Next, we will introduce the implementation principles of the two features brought by the introduction of eBPF, but please note that many processing details will be ignored here.

### Traffic interception

**Outbound traffic**

The figure below shows the interception of outbound traffic. Attach a BPF program to the socket operation connect, and in the program determine whether the current pod is managed by the service mesh, that is, whether it has a sidecar injected, and then modify the destination address to `127.0.0.1` and the destination port to the sidecar's outbound port `15003`. It is not enough to just modify it. The original destination address and port should also be saved in a map, using the socket's cookie as the key.

After the connection with the sidecar is established, the original destination is saved in another map through a program attached to the mount point `sock_ops`, using **local address + port and remote address + port** as the key. When the sidecar accesses the target application later, it obtains the original destination through the `getsockopt` operation on the socket. Yes, a eBPF program is also attached to `getsockopt`, which retrieves the original destination address from the map and returns it.

![](/docs/images/ebpf/outbound-traffic-interception.png)


**Inbound traffic**

For the interception of inbound traffic, the traffic originally intended for the application port is forwarded to the sidecar's inbound port `15003`. There are two cases:

- In the first case, the requester and the service are located on the same node. After the requester's sidecar connect operation is intercepted, the destination port is changed to `15003`.
- In the second case, the requester and the service are located on different nodes. When the handshake packet reaches the service's network namespace, it is intercepted by the BPF program attached to the tc (traffic control) ingress, and the port is modified to `15003`, achieving a functionality similar to DNAT.

### Network communication acceleration

In Kubernetes networks, network packets unavoidably undergo multiple kernel network protocol stack processing. eBPF accelerates network communication by bypassing unnecessary kernel network protocol stack processing and directly exchanging data between two sockets that are peers.

The figure in the traffic interception section shows the sending and receiving trajectories of messages. When the program attached to sock_ops discovers that the connection is successfully established, it saves the socket in a map, using **local address + port and remote address + port** as the key. As the two sockets are peers, their local and remote information is opposite, so when a socket sends a message, it can directly address the peer socket from the map.

This solution also applies to communication between two pods on the same node.

![](/docs/images/ebpf/inbound-traffic-interception.png)


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