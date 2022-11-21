---
title: 「出口」
description: 「サービス メッシュの外部にあるインターネットとサービスへのアクセスを有効にします。」
type: docs
weight: 6
---

# 下り

## インターネットおよびメッシュ外サービスへのアクセスを許可する (エグレス)

このドキュメントでは、インターネットおよびサービス メッシュの外部のサービス (「送信」トラフィックと呼ばれる) へのアクセスを有効にするために必要な手順について説明します。

osm-edge は、メッシュ内のポッドからのすべてのアウトバウンド トラフィックをポッドのサイドカー プロキシにリダイレクトします。 アウトバウンド トラフィックは、次の 2 つのカテゴリに分類できます。

1. メッシュ内トラフィックと呼ばれる、メッシュ クラスタ内のサービスへのトラフィック
2. メッシュ クラスタの外部にあるサービスへのトラフィック。下りトラフィックと呼ばれます。

メッシュ内トラフィックは L7 トラフィック ポリシーに基づいてルーティングされますが、エグレス トラフィックは別の方法でルーティングされ、メッシュ内トラフィック ポリシーの対象にはなりません。 osm-edge は、外部サービスへのアクセスをパススルーとしてサポートし、そのようなトラフィックにフィルタリング ポリシーを適用しません。

## エグレスの設定

Egress を構成するには、次の 2 つのメカニズムがあります。

1. Egress ポリシー API の使用: 外部トラフィックに対するきめ細かいアクセス制御を提供する
2. メッシュ全体のグローバル egress パススルー設定の使用: 設定はオンまたはオフに切り替えられ、メッシュ内のすべてのポッドに影響を与えます。これを有効にすると、メッシュ外の宛先宛てのトラフィックがポッドから送信されます。

## 1. エグレス ポリシーの構成

osm-edge は、その [Egress policy API][1] を使用して、外部エンドポイント宛てのトラフィックのきめ細かなポリシーの構成をサポートしています。 この機能を使用するには、有効になっていない場合は有効にします。

```bash
# Replace osm-system with the namespace where osm-edge is installed
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

さまざまなプロトコルの送信トラフィックをルーティングするためのポリシーを構成する方法については、[Egress policy demo](docs/demos/egress_policy) と [API documentation][1] を参照してください。

## 2. メッシュ全体の Egress パススルーの構成

### 外部宛先へのメッシュ全体の Egress パススルーの有効化

Egress は、osm-edge のインストール中またはインストール後にメッシュ全体で有効にすることができます。 メッシュ全体でエグレスが有効になっている場合、ポッドからのアウトバウンド トラフィックは、それ以外の場合はトラフィックを拒否するメッシュ内トラフィック ポリシーにトラフィックが一致しない限り、ポッドからのエグレスを許可されます。

1. osm-edge インストール中 (デフォルト `osm.enableEgress=false`):

   ```bash
   osm install --set osm.enableEgress=true
   ```

2. osm-edge のインストール後:

    `osm-controller` は、osm メッシュ コントロール プレーン名前空間 (デフォルトでは `osm-system`) の `osm-mesh-config` `MeshConfig` カスタム リソースから出力構成を取得します。 `kubectl patch` を使用して、`osm-mesh-config` リソースで `enableEgress` を `true` に設定します。

   ```bash
   # Replace osm-system with the namespace where osm-edge is installed
   kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":true}}}' --type=merge
   ```

### 外部宛先へのメッシュ全体の Egress パススルーの無効化

egress を有効にするのと同様に、osm-edge のインストール中またはインストール後にメッシュ全体の egress を無効にすることができます。

1. osm-edge インストール中:

   ```bash
   osm install --set osm.enableEgress=false
   ```

2. osm-edge のインストール後:
    `kubectl patch` を使用して、`osm-mesh-config` リソースで `enableEgress` を `false` に設定します。
   ```bash
   # Replace osm-system with the namespace where osm-edge is installed
   kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
   ```

egress を無効にすると、メッシュ内のポッドからのトラフィックは、クラスター外の外部サービスにアクセスできなくなります。

### 使い方

メッシュ全体でエグレスが有効になっている場合、osm-edge コントローラーはメッシュ内のすべての Pipy プロキシ サイドカーをプログラムし、メッシュ内サービスに対応しないアウトバウンドの宛先に一致するワイルドカード ルールを使用します。 このような外部トラフィックに一致するワイルドカード ルールは、トラフィックに L4 または L7 トラフィック ポリシーを適用せずに、トラフィックをそのまま元の宛先にプロキシするだけです。

osm-edge は、基礎となるトランスポートとして TCP を使用するトラフィックの送信をサポートします。 これには生の TCP トラフィック、HTTP、HTTPS、gRPC などが含まれます。

メッシュ全体の送信はグローバル設定であり、不明な宛先へのパススルーとして動作するため、送信トラフィックに対するきめ細かいアクセス制御 (TCP または HTTP ルーティング ポリシーの適用など) は不可能です。

詳細については、[Egress passthrough demo](docs/demos/egress_passthrough) を参照してください。

#### ピピー構成

メッシュでエグレスがグローバルに有効になっている場合、osm-edge コントローラーは各 Pipy プロキシ サイドカーに対して次の構成を発行します。

```json
{
  "Spec": {
    "SidecarLogLevel": "error",
    "Traffic": {
      "EnableEgress": true
    }
  }
}
```

「EnableEgress=true」の Pipy スクリプトは、元の宛先ロジックを使用してリクエストをルーティングし、元の宛先にプロキシします。

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.EgressSpec
