---
title: "出站流量 IP 范围排除"
description: "从 Sidecar 拦截中排除出站流量的 IP 地址范围"
weight: 5
---

This guide demonstrates how outbound IP address ranges can be excluded from being intercepted by osm-edge's proxy sidecar, so as to not subject them to service mesh filtering and routing policies.

本指南演示如何将出站 IP 地址范围从 osm-edge 的代理 sidecar 的拦截中排除，以便它们不受服务网格过滤和路由策略的约束。

## 先决条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 osm-edge。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm`  命令行工具，用于管理服务网格。


## 演示

以下演示展示了一个 HTTP `curl` 客户端 直接使用 IP 地址访向 `httpbin.org` 网站发送 HTTP 请求。我们将显示地禁用出口功能来确保非网格目的地（本演示中的 `httpbin.org`）无法访问 pod 外部。

1. 禁用网格范围的出口直通。

    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
    ```

2. 在 `curl` 命名空间下部署 `curl` 客户端，并将命名空间纳入网格管理。

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    确认 `curl` 客户端 pod 启动并运行。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

3. 获取 `httpbin.org` 网站的公共 IP。为了演示，我们将测试在流量拦截中排除单个 IP 范围。在这个示例中，我们将使用 IP 地址范围 `54.208.105.16/32` 来表示 `54.208.105.16`，在配置和不配置出站 IP 范围排除的情况下发送 HTTP 请求。

    ```console
    $ nslookup httpbin.org
    Server:		1.0.0.1
    Address:	1.0.0.1#53
    
    Name:	httpbin.org
    Address: 3.94.154.124
    Name:	httpbin.org
    Address: 35.173.123.14
    Name:	httpbin.org
    Address: 54.208.105.16
    Name:	httpbin.org
    Address: 34.195.104.96
    Name:	httpbin.org
    Address: 34.227.213.82
    Name:	httpbin.org
    Address: 18.215.122.215
    ```

    > 注意：将 `54.208.105.16` 替换为上面命令行返回的有效 IP 地址。

4. 确认 `curl` 客户端无法发送请求到运行在 `http://54.208.105.16:80` 的 `httpbin.org` 网站。

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://54.208.105.16:80
    curl: (56) Recv failure: Connection reset by peer
    command terminated with exit code 56
    ```

    上述错误是意料之中的，因为默认情况下，出站流量通过运行在 `curl` 客户端 pod 上的 Pipy 代理 sidecar 重定向，并且代理将此流量遵循不允许此流量的服务网格策略。

5. 程序 osm-edge 排除 IP 地址范围 `54.208.105.16/32`。

    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["54.208.105.16/32"]}}}'  --type=merge
    ```

6. 确认网格配置 MeshConfig 已按预期更新。
    ```console
    # 54.208.105.16 is one of the IP addresses of httpbin.org
    $ kubectl get meshconfig osm-mesh-config -n "$osm_namespace" -o jsonpath='{.spec.traffic.outboundIPRangeExclusionList}{"\n"}'
    ["54.208.105.16/32"]
    ```

7. 重启 `curl` 客户端 pod 让出口 IP 地址排除配置生效。这里需要注意，因为流量拦截规则是通过初始化容器在 pod 创建阶段写入的，现有的 pod 必须重启才能生效。

    ```bash
    kubectl rollout restart deployment curl -n curl
    ```

    等待 pod 重启并运行。

8. 确认 `curl` 客户端可以成功发送请求到运行在 `http://54.208.105.16:80` 的 `httpbin.org` 网站。

    ```console
    # 54.208.105.16 is one of the IP addresses for httpbin.org
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://54.208.105.16:80
    HTTP/1.1 200 OK
    Date: Mon, 04 Jul 2022 06:44:28 GMT
    Content-Type: text/html; charset=utf-8
    Content-Length: 9593
    Connection: keep-alive
    Server: gunicorn/19.9.0
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    ```

9. 确认其他没有被排除的 `httpbin.org` IP 地址无法访问。

    ```console
    # 35.173.123.14 is one of the IP addresses for httpbin.org
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://35.173.123.14:80
    curl: (56) Recv failure: Connection reset by peer
    command terminated with exit code 56
    ```