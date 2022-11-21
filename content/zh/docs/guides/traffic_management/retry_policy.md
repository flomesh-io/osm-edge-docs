---
title: "重试"
description: "实现重试来处理短暂的故障"
type: docs
weight: 11
---

# 重试

重试是一种弹性模式，其可以使一个应用能够防御来自客户的暂时性故障。这个可以通过重发因为临时性错误（诸如一个 pod 正在启动）而失败的请求来实现。这个指南描述了如何在 osm-edge 中实现重试策略。


## 配置重试

osm-edge 使用它的 [重试策略 API][1] 来准许在从一个指定的源（服务账户）到一个或者多个目标（服务）的流量上做重试。重试只能应用到 HTTP 流量上。osm-edge 能够为网格中的应用实现重试。

支持如下的重试配置：

- `Per Try Timeout`：即将要进行的重试的超时时间，超过该时间重试被当作失败处理。默认使用全局的路由超时时间。

- `Retry Backoff Base Interval`：对于指数型重试回退的基本间隔。该回退从范围 [0,(2**N-1)B] 中随机选择，这里 N 是重试次数，B 是基本间隔。默认值是 `25ms` 而最大间隔是基本间隔的 10 倍。

- `Number of Retries`：尝试重试的最大次数。默认是 `1`。

- `Retry On`：对一个失败的请求要被重试时指定策略。多个策略能够通过使用 `,` 划分的列表来指定。

如果想了解更多关于配置重试的信息，请参阅[重试策略演示](/docs/demos/retry_policy)和 [API 文档][1]

### 示例
如果从 bookbuyer 服务到 bookstore-v1 服务或者 bookstore-v2 服务的请求收到带 5xx 状态码的响应，那么 bookbuyer 将重试发送 3 遍请求。如果一个尝试的重试耗时超过 3s，它将被认作一个失败的尝试。每次重试在它被尝试[计算](#配置重试)之前有一个延迟周期（回退）。所有重试的回退被限定在 10s。
```yaml
kind: Retry
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: retry
spec:
  source:
    kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
  destinations:
  - kind: Service
    name: bookstore
    namespace: bookstore-v1
  - kind: Service
    name: bookstore
    namespace: bookstore-v2
  retryPolicy:
    retryOn: "5xx"
    perTryTimeout: 3s
    numRetries: 3
    retryBackoffBaseInterval: 1s
```

如果从 bookbuyer 服务到 bookstore-v2 服务的请求收到带状态码 5xx 或者可重试-4xx（409）的响应，那么 bookbuyer 将重试发 5 次请求。如果一个尝试的重试耗时超过 4s，它将被看作一次失败的尝试。每次重试在它被尝试[计算](#配置重试)之前有一个延迟周期（回退）。对所有重试回退被限定在 20ms。
```yaml
kind: Retry
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: retry
spec:
  source:
    kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
  destinations:
  - kind: Service
    name: bookstore
    namespace: bookstore-v2
  retryPolicy:
    retryOn: "5xx,retriable-4xx"
    perTryTimeout: 4s
    numRetries: 5
    retryBackoffBaseInterval: 2ms
```

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.RetrySpec