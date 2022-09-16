---
title: "トラフィックポリシーの設定"
description: "ブックストアアプリケーション間を流れるようにトラフィックを設定する"
type: docs
weight: 3
---

# トラフィックポリシー

## トラフィックポリシーモード

アプリケーションが起動して実行されると、 [permissive traffic policy mode](#permissive-traffic-policy-mode)または [SMI traffic policy mode](#smi-traffic-policy-mode)を使用して相互にやり取りできる。許容なトラフィックポリシーモードでは、アプリケーションサービス間のトラフィックは osm-controller によって自動的に設定され、SMI トラフィックターゲットによって定義されたアクセス制御ポリシーは適用されていない。SMI ポリシーモードでは、SMI アクセス ポリシーとルーティングポリシーの組み合わせを使用して明示的に許可しない限り、すべてのトラフィックがデフォルトで拒否される。

### トラフィックの暗号化

アクセス制御ポリシーを使用しているか、permissive トラフィック ポリシー モードを有効にしているかに関係なく、すべてのトラフィックは mTLS 経由で暗号化される。

### トラフィック ポリシー モードを確認する方法

`osm-mesh-config` `MeshConfig`リソースの enablePermissiveTrafficPolicyMode キーの値を取得して、許容なトラフィックポリシーモードが有効になっているかどうかを確認する。

```bash
# Replace osm-system with osm-controller's namespace if using a non default namespace
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enablePermissiveTrafficPolicyMode}{"\n"}'
# Output:
# false: permissive traffic policy mode is disabled, SMI policy mode is enabled
# true: permissive traffic policy mode is enabled, SMI policy mode is disabled
```

次のセクションでは、 [permissive traffic policy mode](#permissive-traffic-policy-mode)と [SMI Traffic Policy Mode](#smi-traffic-policy-mode)で osm-edge を使用する方法を示す。

## 許容トラフィック ポリシー モード

許容トラフィックポリシーモードでは、メッシュ内のアプリケーション接続は osm-controller によって自動的に設定される。以下の方法で有効化できる。

1. osm` CLIを使ったインストール時。
  ```bash
  osm install --set=osm.enablePermissiveTrafficPolicy=true
  ```

1. コントロールプレーンの名前空間（デフォルトではosm-system）のosm-mesh-configカスタムリソースにパッチを当ててインストール後
  ```bash
  kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
  ```

### osm-edge が許容なトラフィックポリシー モードであることを確認する

続行する前に、[verify the traffic policy mode](#verify-the-traffic-policy-mode)、osm-mesh-config MeshConfig リソースで enablePermissiveTrafficPolicyMode キーが true に設定されていることを確認する。許容なトラフィックポリシーモードを有効にするには、上記のセクションを参照してください。

[Deploy the Bookstore Application](#deploy-the-bookstore-application)の手順では、許容なトラフィックポリシーモードでトラフィックフローを確認するために必要なアプリケーションを既に展開している。 以前デプロイしたブックストアサービスは、 [Deployment's manifest](https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookstore.yaml) で確認できるように、デモ用に bookstore-v1 というIDでエンコードされている。IDは、bookbuyerとbookthiefのUIでどのカウンターをインクリメントするか、bookstoreのUIで表示されるIDを反映している。

bookbuyer, bookthiefのUIで、bookstore v1から買った本と盗んだ本のカウンターがそれぞれインクリメントされるようになった。

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**

UI の `bookstore` にある販売された本のカウンターもインクリメントされるはずだ。

- [http://localhost:8084](http://localhost:8084) - **bookstore**

許容トラフィックポリシーモードが有効であるため、bookbuyerとbookthiefのアプリケーションは、新しく展開されたbookstoreアプリケーションからそれぞれ本を買ったり盗んだりすることができ、それによってSMIトラフィックアクセスポリシーの必要なしにアプリケーション間の接続を可能にする。

このことは、許容なトラフィックポリシーモードを無効にして、書店で購入した本のカウンターがもう増加しないことを確認することで、さらに実証することができる。

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

_注意: 許容トラフィックポリシーモードを無効にすると、SMI トラフィックアクセスモードが暗黙的に有効になる。本のカウンターが増加している場合、そのようなトラフィックを許可するSMIトラフィックアクセスポリシーが以前に適用されたことが原因である可能性がある。_

## SMIトラフィックポリシーモード

SMI トラフィック ポリシーは、次の目的で使用できる。

1. サービスID間のトラフィックアクセスを認可するためのSMIアクセス制御ポリシー
1. アクセス制御ポリシーと関連付けるルーティングルールを定義するSMIトラフィックの仕様ポリシー
1. ウエイトに基づいてクライアント トラフィックを複数のバックエンドに転送する SMI トラフィックスプリットポリシー

次のセクションでは、これらの各ポリシーを活用して、サービス メッシュ内を流れるトラフィックをきめ細かく制御する方法について説明する。続行する前に、[verify the traffic policy mode](#verify-the-traffic-policy-mode)、osm-mesh-config MeshConfig リソースで enablePermissiveTrafficPolicyMode キーが false に設定されていることを確認する。

許容トラフィックポリシーモードを無効にすることで、SMIトラフィックポリシーモードを有効にできる。

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

### SMIアクセスコントロールポリシーをデプロイする

この時点では、アクセス制御ポリシーが適用されていないため、アプリケーションは相互にアクセスできない。bookbuyer、bookthief、bookstore、bookstore-v2のUIにあるカウンターのどれもがインクリメントしていないことを確認してください。

[SMI Traffic Target][https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-access/v1alpha2/traffic-access.md)と[SMI Traffic Specs](https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-specs/v1alpha4/traffic-specs.md) リソースを適用して、通信するアプリケーションのアクセス制御とルーティングのポリシーを定義する。

 SMI TrafficTarget および HTTPRouteGroup ポリシーをデプロイする。

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/access/traffic-access-v1.yaml
```

`bookbuyer` および `bookstore` アプリケーションのカウンタが増加するはずだ。

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8084](http://localhost:8084) - **bookstore**

「bookthief」アプリケーションの場合、カウンターは増加しないことに注意してください。

- [http://localhost:8083](http://localhost:8083) - **bookthief**

これは、デプロイされた SMI トラフィック ターゲット SMI HTTPRouteGroup リソースが、bookbuyer と bookstore との通信のみを許可するためだ。

#### Bookthief アプリケーションがメッシュにアクセスできるようにする

現在、Bookthief アプリケーションは、サービスメッシュ通信への参加を承認されていない。 ここで、TrafficTarget を更新して、「bookthief」が「bookstore」と通信できるようにする。

「spec.sources」に「bookthief」がリストされていない現在のトラフィックターゲット仕様:
```yaml
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: bookstore-v1
  namespace: bookstore
spec:
  destination:
    kind: ServiceAccount
    name: bookstore
    namespace: bookstore
  rules:
  - kind: HTTPRouteGroup
    name: bookstore-service-routes
    matches:
    - buy-a-book
    - books-bought
  sources:
  - kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
```

`spec.sources` で `book thief` を使用して Traffic Target 仕様を更新:

```yaml
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
 name: bookstore-v1
 namespace: bookstore
spec:
 destination:
   kind: ServiceAccount
   name: bookstore
   namespace: bookstore
 rules:
 - kind: HTTPRouteGroup
   name: bookstore-service-routes
   matches:
   - buy-a-book
   - books-bought
 sources:
 - kind: ServiceAccount
   name: bookbuyer
   namespace: bookbuyer
 - kind: ServiceAccount
   name: bookthief
   namespace: bookthief
```

更新された TrafficTarget を適用する。

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/access/traffic-access-v1-allow-bookthief.yaml
```

「bookthief」ウィンドウのカウンターが増え始める。

- [http://localhost:8083](http://localhost:8083) - **bookthief**

bookthiefを許可されたソースとしてリストせずに、元のトラフィックターゲットオブジェクトを適用する。

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/access/traffic-access-v1.yaml
```

「bookthief」ウィンドウのカウンターが増加しなくなる。

- [http://localhost:8083](http://localhost:8083) - **bookthief**

## 次のステップ

[configuring traffic split](/docs/getting_started/traffic_split/)で、サービス間のトラフィックのバランスを取る方法を学ぶ。