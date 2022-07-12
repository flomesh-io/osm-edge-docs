---
title: "安装 osm-edge 控制平面"
description: "这个章节描述了如何在一个 Kubernetes 集群上安装/卸载开放边缘服务网格 (osm-edge)"
type: docs
weight: 2
---

## 先决条件

- Kubernetes 集群，运行 Kubernetes {{< param min_k8s_version >}} 或者更高版本
- [osm-edge CLI](/docs/guides/cli)，或者 [helm 3 CLI](https://helm.sh/docs/intro/install/)，或者 OpenShift `oc` CLI。

### Kubernetes 支持

osm-edge 能够运行在被 osm-edge 版本发布时所支持的 Kubernetes 版本上。当前支持矩阵是：

| 开放边缘服务网格 (osm-edge) | Kubernetes |
| ----------------- | ----------- |
| 1.1               | 1.19 - 1.24 |

### 使用 osm-edge CLI

使用 `osm` CLI 来安装 osm-edge 控制平面到一个 Kubernetes 集群上。

#### osm-edge CLI 和 chart 兼容性

每一个版本的 osm-edge CLI 被设计成只能和与其匹配的 osm-edge Helm chart一起工作。当一些版本偏差存在时，许多操作或许是可以依旧工作的，但是这些情况并没有经过测试，当使用不同版本的 CLI 和flomesh-io/osm-edge时，问题会产生，而且这些问题即便被报告了也可能不会被修复。

#### 运行 CLI

运行 `osm install` 来安装 osm-edge 控制平面。

```console
$ osm install
osm-edge installed successfully in namespace [osm-system] with mesh name [osm]
```

运行 `osm install --help` 以了解更多的选项。

_Note: 通过 CLI 安装的 osm-edge 强制部署唯一一个网格在集群里。osm-edge 安装和管理 CRD，通过添加一个转换网络钩子域到全部的 CRD 来支持多个 API 版本，这个维系了 CRD 到一个指定的 osm-edge 实例。因此，对 osm-edge 的正确操作是被**强烈建议**每个集群只有一个 osm-edge 网格。_

### 使用 Helm CLI

[osm-edge chart](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/charts/osm)能够通过 [Helm CLI](https://helm.sh/docs/intro/install/) 被直接安装。

#### 编辑 values.yaml 文件

可以通过覆盖值文件来配置 osm-edge 安装。

1. 创建一个 [values.yaml 文件](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/values.yaml)的副本（确保使用的针对flomesh-io/osm-edge的版本是要安装的）。关于 values.yaml 中的参数说明，可以参考 [osm-edge Helm Chart 说明文档](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/README.md)。
2. 修改任何想要定制的值。可以忽略所有其他的值。

   - 要查阅哪些值对应到 MeshConfig 设定，请参阅 [osm-edge MeshConfig 文档](/docs/guides/mesh_config)

   - 例如，要设置 MeshConfig 里面的 `logLevel` 域的值为 `info`，保存如下的内容作为 `override.yaml`：
     ```
     osm:
       sidecarLogLevel: info
     ```

#### Helm 安装

然后运行下面的 `helm install` 命令。chart 版本可以在打算安装的 Helm chart 的[这里](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/Chart.yaml#L17)找到。

```console
$ helm install <mesh name> osm --repo https://flomesh-io.github.io/osm-edge --version <chart version> --namespace <osm namespace> --values override.yaml
```

如果首选使用默认的设置，那么省去 `--values` 标志。

运行 `helm install --help` 了解更多选项。

### OpenShift

在 OpenShift 上安装 osm-edge：

1. 启用特权初始化容器，它们能够正确地设置 iptables。在 OpenShift 上，NET_ADMIN 的能力是不够的。
   ```shell
   osm install --set="osm.enablePrivilegedInitContainer=true"
   ```
   - 如果已经安装了 osm-edge，但是没有启用特权初始化容器，那么在 [osm-edge MeshConfig](/docs/guides/mesh_config)里设置 `enablePrivilegedInitContainer` 为 `true`，然后重启网格中的任意的 Pod。
2. 添加 `privileged` [安全上下文限制](https://docs.openshift.com/container-platform/4.7/authentication/managing-security-context-constraints.html)到网格中的每个服务账号。
   - 安装 [oc CLI](https://docs.openshift.com/container-platform/4.7/cli_reference/openshift_cli/getting-started-cli.html)。
   - 添加安全上下文限制到 service account
     ```shell
      oc adm policy add-scc-to-user privileged -z <service account name> -n <service account namespace>
     ```

### Pod 安全策略

**弃用的：从 v0.10.0 开始，在 osm-edge 中 PSP 支持已经被弃用了**

**PSP 支持在 osm-edge 1.0.0 中将被移除**

如果正在一个集群中运行 osm-edge，并启用 PSP，传递 `--set osm.pspEnabled=true` 给 `osm install` 或者 `helm install` CLI 命令。

### 在 osm-edge 中启用 Reconciler

如果打算在 osm-edge 中启用 Reconciler，传递 `--set osm.enableReconciler=true` 给 `osm install` 或者 `helm install` CLI 命令。关于 Reconciler 的更多信息，请参阅 [Reconciler 指南](/docs/guides/reconciler)。

## 检查 osm-edge 组件

一些组件将被默认安装。检查它们通过使用下面的 `kubectl` 命令：

```console
# Replace osm-system with the namespace where osm-edge is installed
$ kubectl get pods,svc,secrets,meshconfigs,serviceaccount --namespace osm-system
```

一些集群范围（非命名空间组件）也将被安装。检查它们使用下面的 `kubectl` 命令：

```console
kubectl get clusterrolebinding,clusterrole,mutatingwebhookconfiguration,validatingwebhookconfigurations -l app.kubernetes.io/name=openservicemesh.io
```

在底层，`osm` 在控制平面所在的命名空间里通过 [Helm](https://helm.sh) 库来创建一个 Helm `release` 对象。Helm `release` 名称是 mesh-name。`helm` CLI 也能够被用来检查更详细的已安装 Kubernetes 清单。去往 https://helm.sh 了解安装 Helm 的指令。

```console
# Replace osm-system with the namespace where osm-edge is installed
$ helm get manifest osm --namespace osm-system
```

## 下一步

现在，osm-edge 控制平面启动并运行了，[添加服务](/docs/guides/app_onboarding/)到网格吧。
