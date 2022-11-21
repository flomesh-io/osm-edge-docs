---
title: "宽松模式"
description: "宽松流量策略模式"
type: docs
weight: 2
---

# 宽松流量策略模式
osm-edge 中的宽松流量策略模式是绕过 [SMI][1] 流量访问策略执行的模式。在这种模式下，osm-edge 会自动发现属于服务网格一部分的服务，并在每个 Pipy 代理 sidecar 上编写流量策略规则，以便能够与这些服务进行通信。

## 何时使用宽松流量策略模式

由于宽松流量策略模式绕过 [SMI][1] 流量访问策略执行，因此它适用于服务网格内的应用程序之间的连接应该像应用程序注册到网格之前一样流动时使用。该模式适用于无法明确定义应用间连接的流量访问策略的环境。

启用宽松流量策略模式的一个常见用例是在不中断应用程序连接的情况下支持将应用程序逐步加入网格。应用服务之间的流量路由是由 osm-edge 控制器通过服务发现自动建立的。在每个 Pipy 代理 sidecar 上设置通配符流量策略，以允许流量流向网格内的服务。

允许流量策略模式的替代方案是 SMI 流量策略模式，其中应用程序之间的流量默认被拒绝，并且显式 SMI 流量策略是允许应用程序连接所必需的。当需要执行策略时，必须使用 SMI 流量策略模式。

## 配置宽松流策略模式

可以在安装 osm-edge 时或安装 osm-edge 后启用或禁用宽松流量策略模式。

### 启用宽松流量策略模式

启用宽松流量策略模式会禁用 SMI 流量策略模式。

在 osm-edge 安装时通过 `--set` 标识：

```bash
osm install --set osm.enablePermissiveTrafficPolicy=true
```

在 osm-edge 安装之后：

```bash
# Assumes osm-edge is installed in the osm-system namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
```

### 禁用宽松流量策略模式

禁用宽松流量策略模式会启动 SMI 流量策略模式。

在 osm-edge 安装时通过 `--set` 标识：

```bash
osm install --set osm.enablePermissiveTrafficPolicy=false
```

在 osm-edge 安装之后：

```bash
# Assumes osm-edge is installed in the osm-system namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

## 工作原理

启用宽松流量策略模式后，osm-edge 控制器会发现属于网格的所有服务，并在每个 Pipy 代理 sidecar 上编写通配符流量路由规则，以访问网格中的所有其他服务。此外，与服务相关联的每个代理前端工作负载都配置为接受以服务为目标的所有流量。根据服务的应用协议（HTTP、TCP、gRPC 等），在 Pipy sidecar 上配置适当的流量路由规则，以允许该特定类型的所有流量。

参考[宽松流量策略模式演示](docs/demos/permissive_traffic_mode) 了解更多。

### Pipy 配置

在宽松模式下，osm-edge 控制器为客户端应用程序编写通配符路由以与服务通信。 以下是来自 `curl` 和 `httpbin` sidecar 代理的 Pipy 入站和出站过滤器和路由配置片段。

1. `curl` 客户端 pod 上的 Outbound Pipy 配置：

     `httpbin` 服务对应的出站 HTTP 过滤器链：
         
    ```json
     {
      "Outbound": {
        "TrafficMatches": {
          "14001": [
            {
              "DestinationIPRanges": [
                "10.43.103.59/32"
              ],
              "Port": 14001,
              "Protocol": "http",
              "HttpHostPort2Service": {
                "httpbin": "httpbin.app.svc.cluster.local",
                "httpbin.app": "httpbin.app.svc.cluster.local",
                "httpbin.app.svc": "httpbin.app.svc.cluster.local",
                "httpbin.app.svc.cluster": "httpbin.app.svc.cluster.local",
                "httpbin.app.svc.cluster.local": "httpbin.app.svc.cluster.local",
                "httpbin.app.svc.cluster.local:14001": "httpbin.app.svc.cluster.local",
                "httpbin.app.svc.cluster:14001": "httpbin.app.svc.cluster.local",
                "httpbin.app.svc:14001": "httpbin.app.svc.cluster.local",
                "httpbin.app:14001": "httpbin.app.svc.cluster.local",
                "httpbin:14001": "httpbin.app.svc.cluster.local"
              },
              "HttpServiceRouteRules": {
                "httpbin.app.svc.cluster.local": {
                  ".*": {
                    "Headers": null,
                    "Methods": null,
                    "TargetClusters": {
                      "app/httpbin|14001": 100
                    },
                    "AllowedServices": null
                  }
                }
              },
              "TargetClusters": null,
              "AllowedEgressTraffic": false,
              "ServiceIdentity": "default.app.cluster.local"
            }
          ]
        }
      }
    }
    ```

    出站路由配置：
    
    ```json
    "HttpServiceRouteRules": {
            "httpbin.app.svc.cluster.local": {
              ".*": {
                "Headers": null,
                "Methods": null,
                "TargetClusters": {
                  "app/httpbin|14001": 100
                },
                "AllowedServices": null
              }
            }
          }
    ```

2. `httpbin` 服务 pod 的入站 Pipy 配置：

    `httpbin` 服务对应的入站 HTTP 过滤器链：
    
    ```json
    {
      "Inbound": {
        "TrafficMatches": {
          "14001": {
            "SourceIPRanges": null,
            "Port": 14001,
            "Protocol": "http",
            "HttpHostPort2Service": {
              "httpbin": "httpbin.app.svc.cluster.local",
              "httpbin.app": "httpbin.app.svc.cluster.local",
              "httpbin.app.svc": "httpbin.app.svc.cluster.local",
              "httpbin.app.svc.cluster": "httpbin.app.svc.cluster.local",
              "httpbin.app.svc.cluster.local": "httpbin.app.svc.cluster.local",
              "httpbin.app.svc.cluster.local:14001": "httpbin.app.svc.cluster.local",
              "httpbin.app.svc.cluster:14001": "httpbin.app.svc.cluster.local",
              "httpbin.app.svc:14001": "httpbin.app.svc.cluster.local",
              "httpbin.app:14001": "httpbin.app.svc.cluster.local",
              "httpbin:14001": "httpbin.app.svc.cluster.local"
            },
            "HttpServiceRouteRules": {
              "httpbin.app.svc.cluster.local": {
                ".*": {
                  "Headers": null,
                  "Methods": null,
                  "TargetClusters": {
                    "app/httpbin|14001|local": 100
                  },
                  "AllowedServices": null
                }
              }
            },
            "TargetClusters": null,
            "AllowedEndpoints": null
          }
        }
      }
    }
    ```

    入站路由配置：
    
    ```json
    "HttpServiceRouteRules": {
      "httpbin.app.svc.cluster.local": {
        ".*": {
          "Headers": null,
          "Methods": null,
          "TargetClusters": {
            "app/httpbin|14001|local": 100
          },
          "AllowedServices": null
        }
      }
    }
    ```


[1]: https://smi-spec.io/
