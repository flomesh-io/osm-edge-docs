---
title: "Permissive Mode"
description: "Permissive Traffic Policy Mode"
weight: 2
---

# Permissive Traffic Policy Mode
Permissive traffic policy mode in osm-edge is a mode where [SMI][1] traffic access policy enforcement is bypassed. In this mode, osm-edge automatically discovers services that are a part of the service mesh and programs traffic policy rules on each Pipy proxy sidecar to be able to communicate with these services.

## When to use permissive traffic policy mode
Since permissive traffic policy mode bypasses [SMI][1] traffic access policy enforcement, it is suitable for use when connectivity between applications within the service mesh should flow as before the applications were enrolled into the mesh. This mode is suitable in environments
where explicitly defining traffic access policies for connectivity between applications is not feasible.

A common use case to enable permissive traffic policy mode is to support gradual onboarding of applications into the mesh without breaking application connectivity. Traffic routing between application services is automatically set up by osm-edge controller through service discovery. Wildcard traffic policies are set up on each Pipy proxy sidecar to allow traffic flow to services within the mesh.

The alternative to permissive traffic policy mode is SMI traffic policy mode, where traffic between applications is denied by default and explicit SMI traffic policies are necessary to allow application connectivity. When policy enforcement is necessary, SMI traffic policy mode must be used instead.

## Configuring permissive traffic policy mode
Permissive traffic policy mode can be enabled or disabled at the time of osm-edge install, or after osm-edge has been installed.

### Enabling permissive traffic policy mode

Enabling permissive traffic policy mode implicitly disables SMI traffic policy mode.

During osm-edge install using the `--set` flag:
```bash
osm install --set osm.enablePermissiveTrafficPolicy=true
```

After osm-edge has been installed:
```bash
# Assumes osm-edge is installed in the osm-system namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
```

### Disabling permissive traffic policy mode

Disabling permissive traffic policy mode implicitly enables SMI traffic policy mode.

During osm-edge install using the `--set` flag:
```bash
osm install --set osm.enablePermissiveTrafficPolicy=false
```

After osm-edge has been installed:
```bash
# Assumes osm-edge is installed in the osm-system namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

## How it works
When permissive traffic policy mode is enabled, osm-edge controller discovers all services that are a part of the mesh and programs wildcard traffic routing rules on each Pipy proxy sidecar to reach every other service in the mesh. Additionally, each proxy fronting workloads that are associated with a service is configured to accept all traffic destined to the service. Depending on the application protocol of the service (HTTP, TCP, gRPC etc.), appropriate traffic routing rules are configured on the Pipy sidecar to allow all traffic for that particular type.

Refer to the [Permissive traffic policy mode demo](/docs/demos/permissive_traffic_mode) to learn more.

### Pipy configurations
In permissive mode, osm-edge controller programs wildcard routes for client applications to communicate with services. Following are the Pipy inbound and outbound filter and route configuration snippets from the `curl` and `httpbin` sidecar proxies.

1. Outbound Pipy configuration on the `curl` client pod:

    Outbound HTTP filter chain corresponding to the `httpbin` service:
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

    Outbound route configuration:
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

2. Inbound Pipy configuration on the `httpbin` service pod:

    Inbound HTTP filter chain corresponding to the `httpbin` service:
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

    Inbound route configuration:
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
