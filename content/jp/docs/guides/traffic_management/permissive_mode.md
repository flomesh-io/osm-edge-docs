---
title: 「寛容モード」
description: 「許容トラフィック ポリシー モード」
type: docs
weight: 2
---

# 許容トラフィック ポリシー モード
osm-edge の許容トラフィック ポリシー モードは、[SMI][1] トラフィック アクセス ポリシーの適用がバイパスされるモードです。 このモードでは、osm-edge はサービス メッシュの一部であるサービスを自動的に検出し、これらのサービスと通信できるように各 Pipy プロキシ サイドカーでトラフィック ポリシー ルールをプログラムします。

## permissive トラフィック ポリシー モードを使用する場合
permissive トラフィック ポリシー モードは [SMI][1] トラフィック アクセス ポリシーの適用をバイパスするため、サービス メッシュ内のアプリケーション間の接続が、アプリケーションがメッシュに登録される前と同じように流れる必要がある場合に使用するのに適しています。このモードは環境に適しています
アプリケーション間の接続のためのトラフィック アクセス ポリシーを明示的に定義することは現実的ではありません。

permissive トラフィック ポリシー モードを有効にする一般的な使用例は、アプリケーションの接続を中断することなく、メッシュへのアプリケーションの段階的なオンボーディングをサポートすることです。アプリケーション サービス間のトラフィック ルーティングは、サービス ディスカバリを通じて osm-edge コントローラによって自動的に設定されます。各 Pipy プロキシ サイドカーでワイルドカード トラフィック ポリシーが設定され、メッシュ内のサービスへのトラフィック フローが許可されます。

permissive トラフィック ポリシー モードに代わるものは SMI トラフィック ポリシー モードです。このモードでは、アプリケーション間のトラフィックはデフォルトで拒否され、アプリケーション接続を許可するには明示的な SMI トラフィック ポリシーが必要です。ポリシーの適用が必要な場合は、代わりに SMI トラフィック ポリシー モードを使用する必要があります。

## permissive トラフィック ポリシー モードの構成
許容トラフィック ポリシー モードは、osm-edge のインストール時、または osm-edge のインストール後に有効または無効にできます。

### permissive トラフィック ポリシー モードの有効化

permissive トラフィック ポリシー モードを有効にすると、暗黙的に SMI トラフィック ポリシー モードが無効になります。

`--set` フラグを使用した osm-edge インストール中:
```bash
osm install --set osm.enablePermissiveTrafficPolicy=true
```

osm-edge のインストール後:
```bash
# Assumes osm-edge is installed in the osm-system namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
```

### permissive トラフィック ポリシー モードの無効化

permissive トラフィック ポリシー モードを無効にすると、暗黙的に SMI トラフィック ポリシー モードが有効になります。

`--set` フラグを使用した osm-edge インストール中:
```bash
osm install --set osm.enablePermissiveTrafficPolicy=false
```

osm-edge のインストール後:
```bash
# Assumes osm-edge is installed in the osm-system namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

## 使い方
permissive トラフィック ポリシー モードが有効な場合、osm-edge コントローラーはメッシュの一部であるすべてのサービスを検出し、各 Pipy プロキシ サイドカーでワイルドカード トラフィック ルーティング ルールをプログラムして、メッシュ内の他のすべてのサービスに到達します。 さらに、サービスに関連付けられている各プロキシ フロンティング ワークロードは、サービス宛てのすべてのトラフィックを受け入れるように構成されています。 サービスのアプリケーション プロトコル (HTTP、TCP、gRPC など) に応じて、適切なトラフィック ルーティング ルールが Pipy サイドカーで構成され、その特定のタイプのすべてのトラフィックが許可されます。

詳細については、[Permissive traffic policy mode demo](docs/demos/permissive_traffic_mode) を参照してください。

### ピピー構成
Permissive モードでは、osm-edge コントローラーは、クライアント アプリケーションがサービスと通信するためのワイルドカード ルートをプログラムします。 以下は、`curl` および `httpbin` サイドカー プロキシからの Pipy インバウンドおよびアウトバウンド フィルタおよびルート構成スニペットです。

1. `curl` クライアント ポッドでのアウトバウンド Pipy 設定:
 `httpbin` サービスに対応するアウトバウンド HTTP フィルタ チェーン:
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

    アウトバウンド ルートの構成:
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

2. 「httpbin」サービス ポッドでのインバウンド Pipy 構成:

     `httpbin` サービスに対応するインバウンド HTTP フィルタ チェーン:
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

    インバウンド ルートの構成:
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
