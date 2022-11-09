---
title: "宽松流量策略模式"
description: "宽松流量策略模式故障排查"
type: docs
weight: 2
---

# 宽松流量策略模式故障排查

## 当宽松流量策略模式没有按预期工作时

### 1. 确认宽松流量策略模式已开启

在自定义资源 `osm-mesh-config` 中，通过检查 `enablePermissiveTrafficPolicyMode` 配置字段，来确认宽松流量策略模式是否启用。`osm-mesh-config` MeshConfig 位于 osm-edge 控制平面所在的命名空间（默认为`osm-system`）

```console
# 如果宽松流量策略模式已启用，则返回 true
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enablePermissiveTrafficPolicyMode}{"\n"}'
true
```

上面的命令一定会返回布尔型的字符串（`true` 或者 `false`），表示宽松流量策略模式是否启用

### 2. 检查 osm-edge 控制器日志中的错误

```bash
# 如果 osm-controller 被部署到 osm-system 命名空间中
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

日志当中，`level` 字段被设置为 `error` 的消息会记录错误信息
```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. 确认 Pipy 的配置

使用 `osm verify connectivity` 命令来验证 pod 能够使用 Kubernetes 服务来通信。

例如，要验证命名空间 `curl` 中的 pod `curl-7bb5845476-zwxbt` 是否能够定向流量到使用 `httpbin` Kubernetes 服务的命名空间 `httpbin` 中的 pod `httpbin-69dc7d545c-n7pjb`：

```console
$ osm verify connectivity --from-pod curl/curl-7bb5845476-zwxbt --to-pod httpbin/httpbin-69dc7d545c-n7pjb --to-service httpbin
---------------------------------------------
[+] Context: Verify if pod "curl/curl-7bb5845476-zwxbt" can access pod "httpbin/httpbin-69dc7d545c-n7pjb" for service "httpbin/httpbin"
Status: Success

---------------------------------------------
```

当验证成功后，输出中的 `Status` 字段将显示 `Success`。