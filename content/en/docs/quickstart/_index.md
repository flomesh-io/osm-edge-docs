---
title: "Quick Start"
description: "Try osm-edge in 5 minutes"
type: docs
weight: 2
---

# osm-edge quick start guide

This guide shows how to download, install, and run osm-edge, deploy a demo application, and complete SMI standard functionality like link encryption, access control, and traffic splitting in less than 5 minutes. This demo assumes you are running Ubuntu 21 on x86 architecture, running the k3s version 'V1.23.8 + K3S1'. For more version and platform support, please refer to the complete [Beginner's Guide](/docs/getting_started/).

## Pre-requisites

Running Kubernetes cluster. If you don't have one, you can use below script to install k3s:

```bash
export INSTALL_K3S_VERSION=v1.23.8+k3s1
curl -sfL https://get.k3s.io | sh -s - --disable traefik --write-kubeconfig-mode 644 --write-kubeconfig ~/.kube/config
```

> Minimum Kubernetes version supported by osm-edge is {{< param min_k8s_version >}}

## Download and install osm-edge CLI


### GNU/Linux

Download the 64-bit GNU/Linux or macOS binary of osm-edge {{< param osm_edge_version >}}:

```bash
system=$(uname -s | tr '[:upper:]' '[:lower:]')
arch=$(uname -m)
release={{< param osm_edge_version >}}
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

### macOS

Download the 64-bit macOS binaries for osm-edge {{< param osm_edge_version >}}

```bash
system=$(uname -s | tr "[:upper:]" "[:lower:]")
arch=$(uname -m)
release={{< param osm_edge_version >}}
curl -L https://github.com/flomesh-io/osm-edge/releases/download/$release/osm-edge-$release-$system-$arch.tar.gz | tar -vxzf -
./$system-$arch/osm version
cp ./$system-$arch/osm /usr/local/bin/
```

## Install osm-edge on Kubernetes cluster

> Below command installs and enable [Prometheus](https://github.com/prometheus/prometheus),[Grafana](https://github.com/grafana/grafana), and [Jaeger](https://github.com/jaegertracing/jaeger)

```bash
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.deployPrometheus=true \
    --set=osm.deployGrafana=true \
    --set=osm.deployJaeger=true \
    --set=osm.tracing.enable=true
```
## Deploy Applications

In this section we will deploy 5 different Pods, and we will apply policies to control the traffic between them.

- `bookbuyer` is an HTTP client making requests to `bookstore`. This traffic is **permitted**.
- `bookthief` is an HTTP client and much like `bookbuyer` also makes HTTP requests to `bookstore`. This traffic should be **blocked**.
- `bookstore` is a server, which responds to HTTP requests. It is also a client making requests to the `bookwarehouse` service. This traffic is **permitted**.
- `bookwarehouse` is a server and should respond only to `bookstore`. Both `bookbuyer` and `bookthief` should be blocked.
- `mysql` is a MySQL database only reachable by `bookwarehouse`.

Use below script to install:

```bash
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
osm namespace add bookstore bookbuyer bookthief bookwarehouse
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookbuyer.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookthief.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookstore.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookwarehouse.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/mysql.yaml
```

Expose the GUI ports of each service, so that with a browser we can access these ports of demo application.

```bash
git clone https://github.com/flomesh-io/osm-edge.git -b {{< param osm_branch >}}
cd osm-edge
cp .env.example .env
./scripts/port-forward-all.sh #可以忽略错误信息
```

In a browser, open the following URL.

_Note: If you need to access from the host, you need to replace `localhost` with the IP address of the virtual machine; or run the `port-forward-all.sh` script on the host. _

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**
- [http://localhost:8084](http://localhost:8084) - **bookstore**


## Access Control

By installing osm-edge with the above command, all services are without access control (permissive traffic policy mode), or all access is allowed. The situation when there is no access control can be seen by looking at the growth in the number of books counts per service in the browser.

The counts in the `bookbuyer`, `bookthief` UI correspond to the number of books purchased and stolen, respectively, while in `bookstore-v1` these should be increasing by.

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**

The count for book sales in the `bookstore` UI should also be increasing.

- [http://localhost:8084](http://localhost:8084) - **bookstore**

The following demonstrates denying access to the `bookstore` service by disabling the permissive traffic policy mode.

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

You will see that the count is no longer increasing.

Execute below command to allow `bookbuyer` privileges to access `bookstore`:

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/access/traffic-access-v1.yaml
```

Here we go back to the `bookbuyer` and `bookstore` UI and see that the count resumes increasing while the count for the `bookthief` UI remains stopped.

With access control, we have successfully prevented `bookthief` from stealing books from `bookstore`, while normal purchases are unaffected.

## Observability

### Metrics

Use below command to enable namespace metrics generation and capturing, or else metrics generated by Pods won't be gathered.

```shell
osm metrics enable --namespace "bookstore,bookbuyer,bookthief,bookwarehouse"
```

After running port-forwarding script, open url `http://localhost:3000` in browser to access Grafan console. Dashboard default username and passwords are `admin`, `admin`.

osm-edge has several built-in dashboards to provide visualization of metrics in the control plane and data plane. For example, the following figure shows the metrics of pod `http://localhost:3000` of the `bookthief` service accessing other `services`.

![image](https://user-images.githubusercontent.com/2224492/180593501-d73dbf11-40a8-4fe9-9422-ea931da2927f.png)

The following figure shows the metrics of `bookthief` accessing other `services` at the granularity of `deployment`. The difference from the previous figure is that if `bookthief` has multiple replicas, the aggregate data for all replicas is shown here: !

![image](https://user-images.githubusercontent.com/2224492/180593509-9a852bf1-e7e7-4534-9c57-06cf1c890ee3.png)

The next metrics for the osm-edge component, and for the mesh base information are shown here.

![image](https://user-images.githubusercontent.com/2224492/180593512-0ac33a0e-2b7a-4e66-b499-f196b5dd729b.png)

### Tracing

Jaeger's dashboard can be accessed by typing `http://localhost:16686/search` in your browser: !

![image](https://user-images.githubusercontent.com/2224492/180593520-64b0d2d1-1346-47ac-aab8-a9eaae9f8950.png)

The dashboard allows you to look up service-related tracing information: !

![image](https://user-images.githubusercontent.com/2224492/180593525-3bc844c4-f950-48f6-9d72-ff98dc82aa2c.png)

Show service topology diagram.

![image](https://user-images.githubusercontent.com/2224492/180593530-8d0ed18f-0cac-495f-985f-04feb863ec6d.png)

### Logging

The osm-edge control plane outputs diagnostic logs to the standard output for service mesh management, and the output of logging information can be controlled by adjusting the level of logging. The logs output to the standard output can be aggregated and stored by the log collection tool.

## Uninstall Service Mesh

To uninstall all resources associated with osm-edge after completing the quick experience with osm-edge, you will need to delete these sample applications and associated SMI resources and uninstall the osm-edge control plane and cluster-wide osm-edge resources.

To delete the sample applications.

```shell
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

Uninstall the control plane.

```shell
osm uninstall mesh
```