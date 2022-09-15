---
title: 「アプリケーション プロトコルの選択」
title: 「アプリケーション プロトコルの選択」
type: docs
---

# アプリケーション プロトコルの選択

osm-edge は、`HTTP`、`TCP`、`gRPC` などのさまざまなアプリケーション プロトコルを別々にルーティングできます。 次のガイドでは、サービス ポートを構成して、トラフィックのフィルタリングとルーティングに使用するアプリケーション プロトコルを指定する方法について説明します。

## アプリケーション プロトコルの構成

Kubernetes サービスは、1 つ以上のポートを公開します。 サービスを実行しているアプリケーションによって公開されるポートは、HTTP、TCP、gRPC などの特定のアプリケーション プロトコルを提供できます。osm-edge はさまざまなアプリケーション プロトコルのトラフィックを異なる方法でフィルタリングおよびルーティングするため、Kubernetes サービス オブジェクトの構成は、 osm-edge サービス ポートに向けられたトラフィックをどのようにルーティングする必要があるか。

サービスのポートによって提供されるアプリケーション プロトコルを決定するために、osm-edge は、サービスのポートの「appProtocol」フィールドが設定されていることを期待します。

osm-edge は、サービス ポート用に次のアプリケーション プロトコルをサポートします。
1. `http`: HTTP ベースのトラフィックのフィルタリングとルーティング用
1. `tcp`: TCP ベースのトラフィックのフィルタリングとルーティング用
1. `tcp-server-first`: サーバーが mySQL、PostgreSQL などのクライアントとの通信を開始するトラフィックの TCP ベースのフィルタリングとルーティング用
1. `gRPC`: gRPC トラフィックの HTTP2 ベースのフィルタリングとルーティング用

説明されているアプリケーション プロトコルの設定は、SMI および Permissive トラフィック ポリシー モードの両方に適用できます。

### 例

次の SMI トラフィック アクセスおよびトラフィック仕様ポリシーを検討してください。
- TCP トラフィックが許可されるポートを指定する「tcp-route」という名前の「TCPRoute」リソース。
- HTTP トラフィックを許可する HTTP ルートを指定する「http-route」という名前の「HTTPRouteGroup」リソース。
- サービス アカウント「sa-2」のポッドが、指定された TCP および HTTP ルールのサービス アカウント「sa-1」のポッドにアクセスできるようにする、「test」という名前の「TrafficTarget」リソース。

```yaml
kind: TCPRoute
metadata:
  name: tcp-route
spec:
  matches:
    ports:
    - 8080
---
kind: HTTPRouteGroup
metadata:
  name: http-route
spec:
  matches:
  - name: version
    pathRegex: "/version"
    methods:
    - GET
---
kind: TrafficTarget
metadata:
  name: test
  namespace: default
spec:
  destination:
    kind: ServiceAccount
    name: sa-1 # There are 2 services under this service account:  service-1 and service-2
    namespace: default
  rules:
  - kind: TCPRoute
    name: tcp-route
  - kind: HTTPRouteGroup
    name: http-route
  sources:
  - kind: ServiceAccount
    name: sa-2
    namespace: default
```

Kubernetes サービス リソースは、「appProtocol」フィールドを使用して、サービスのポートによって提供されるアプリケーション プロトコルを明示的に指定する必要があります。

「http」アプリケーション トラフィックを処理するサービス アカウント「sa-1」のポッドによってバッキングされるサービス「service-1」は、次のように定義する必要があります。

```yaml
kind: Service
metadata:
  name: service-1
  namespace: default
spec:
  ports:
  - port: 8080
    name: some-port
    appProtocol: http
```

生の「tcp」アプリケーション トラフィックを提供するサービス アカウント「sa-1」のポッドによってサポートされるサービス「service-2」は、次のように定義する必要があります。

```yaml
kind: Service
metadata:
  name: service-2
  namespace: default
spec:
  ports:
  - port: 8080
    name: some-port
    appProtocol: tcp
```
