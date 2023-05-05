---
title: "Quick Start on OpenShift"
description: "How to quickly setup osm-edge on Redhat OpenShift"
weight: 2
---

If you already have running OpenShift platform, follow these steps to get started quickly.

## Prerequisites

- Openshift version 4.6 or above
- The [osm-edge CLI](/docs/guides/operating/cli) or the [helm 3 CLI](https://helm.sh/docs/intro/install/) or the OpenShift `oc` CLI.


### Helm Chart Repository

OpenShift starting with version 4.8 comes with Helm Chart Repository installed, and if you are using an older version of OpenShift, you can add the repo via helm command:

```
# curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64 -o /usr/local/bin/helm

# chmod +x /usr/local/bin/helm

$ helm repo add openshift-helm-charts https://charts.openshift.io/
```

To install the repo to be used from the OpenShift console run the following command as and OpenShift admin:

```
$ oc apply -f https://charts.openshift.io/openshift-charts-repo.yaml
```

### The Permissions

The account to deploy the charts need to be bound to clusterrole `cluster-admin`, you could achieve by the command:

```
oc adm policy add-cluster-role-to-user cluster-admin <your OCP account name>
```

## Installation

### Install via Helm Cli

Run the following `helm` command to install `osm-edge`

```
helm install \
        --devel \
        --namespace osm-system \
        --create-namespace \
        --set=osm.controllerLogLevel=warn \
        osm-edge \
        openshift-helm-charts/flomesh-osm-edge
```


### Install via OpenShift Console

1. Create project named `osm-system`

2. Find and search `osm-edge` in the Developer Catalog

   ![](https://user-images.githubusercontent.com/10077630/207877795-65e03207-3023-41d2-ada9-49cb4b955275.png)

3. Before starting install the chart, switch configuration style to `YAML view`, as the JSON Schema version is much newer than OpenShift console supported version. You could change the configs as well if needed.
   
   ![](https://user-images.githubusercontent.com/10077630/207879041-b6773358-2b18-4a63-8375-d730ef89840b.png)

4. Click `Install` to start installation and wait the pods to be ready.

   ![](https://user-images.githubusercontent.com/10077630/207879584-0fb0c127-1a40-4d06-b5b4-a8cc5c85f2da.png)

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
