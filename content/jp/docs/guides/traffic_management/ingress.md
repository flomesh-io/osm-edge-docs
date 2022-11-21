---
title: 「イングレス」
description: 「Ingress を使用してクラスター内のサービスへの外部アクセスを管理する」
type: docs
weight: 5
---

# 進入

## Ingress を使用してクラスター内のサービスへの外部アクセスを管理する

Ingress とは、クラスター内のサービス (通常は HTTP/HTTPS サービス) への外部アクセスの管理を指します。 osm-edge のイングレス機能により、クラスター管理者とアプリケーション所有者は、イングレスの実行に使用されるメカニズムに応じた一連のルールを使用して、サービス メッシュ外部のクライアントからサービス メッシュ バックエンドにトラフィックをルーティングできます。

## IngressBackend API

osm-edge は、その [IngressBackend API][1] を活用して、信頼できるソースからのイングレス トラフィックを受け入れるようにバックエンド サービスを構成します。この仕様では、使用されるプロトコル (HTTP または HTTPS) に応じて、特定のバックエンドがイングレス トラフィックを承認する方法を構成できます。バックエンド プロトコルが「http」の場合、指定されたソースの種類は次のいずれかである必要があります。バックエンドに接続します。バックエンド プロトコルが「https」の場合、指定されるソースは、バックエンドが認証するクライアントの証明書にエンコードされたサブジェクト代替名 (SAN) を定義する「AuthenticatedPrincipal」の種類である必要があります。 `Service` または `IPRange` の種類を持つソースは、`https` バックエンドのオプションであり、指定された場合、クライアントはその `AuthenticatedPrincipal` 値に加えてソースと一致する必要があることを意味します。 `https` バックエンドの場合、クライアント証明書の検証はデフォルトで実行され、バックエンドの `tls` フィールドで `skipClientCertValidation: true` を設定することで無効にできます。 「IngressBackend」構成の「backend」サービスの「port.number」フィールドは、Kubernetes サービスの「targetPort」に対応している必要があります。

`IngressBackend` 構成のソースの `Kind` が `Service` に設定されている場合、osm-edge コントローラーはそのサービスのエンドポイントを検出しようとすることに注意してください。 osm-edge がサービスのエンドポイントを検出できるようにするには、サービスが存在する名前空間が監視対象の名前空間である必要があります。 以下を使用して名前空間を監視できるようにします。

```bash
kubectl label ns <namespace> openservicemesh.io/monitored-by=<mesh name>
```

### 例

次の IngressBackend 構成は、トラフィックの発信元が「default」名前空間の「myapp」サービスのエンドポイントである場合にのみ、「test」名前空間のポート「80」で「foo」サービスへのアクセスを許可します。
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http
  sources:
    - kind: Service
      namespace: default
      name: myapp
```

次の IngressBackend 構成は、トラフィックの発信元が CIDR 範囲「10.0.0.0/8」に属する IP アドレスを持っている場合にのみ、「test」名前空間のポート「80」で「foo」サービスへのアクセスを許可します。
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http
  sources:
    - kind: IPRange
      name: 10.0.0.0/8
```

次の IngressBackend 構成は、トラフィックの発信元が「TLS」でトラフィックを暗号化し、サブジェクト代替名 (SAN) の「クライアント」を持っている場合にのみ、「test」名前空間のポート「80」で「foo」サービスへのアクセスを許可します。 クライアント証明書にエンコードされた default.svc.cluster.local`:
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80
        protocol: https # https implies TLS
      tls:
        skipClientCertValidation: false # mTLS (optional, default: false)
  sources:
    - kind: AuthenticatedPrincipal
      name: client.default.svc.cluster.local
```

`http` および `https` バックエンドの `IngressBackend` 構成がどのように見えるかを理解するには、次のセクションを参照してください。

## Ingress を実行するための選択肢

osm-edge は、次のセクションで説明するイングレスを使用してメッシュ サービスを外部に公開するための複数のオプションをサポートしています。 osm-edge は、Contour および OSS Nginx でテストされています。これらは、メッシュの外部にインストールされ、メッシュに参加するための証明書でプロビジョニングされたイングレス コントローラーと連携します。

> 注: Nginx Plus との osm-edge 統合は、Kubernetes シークレットから自己署名 mTLS 証明書を取得するために完全にテストされていません。 ただし、Nginx Plus または任意のイングレスを組み込む別の方法は、それをメッシュにインストールして、メッシュに参加できるようにする Pipy サイドカーが注入されるようにすることです。 Pipy サイドカーをバイパスするには、80 や 443 などの追加のインバウンド ポートを許可する必要がある場合があります。

### 1. FSM イングレス コントローラーとゲートウェイの使用

[FSM](https://github.com/flomesh-io/fsm) イングレス コントローラーとエッジ プロキシを使用することは、osm-edge マネージド サービス メッシュでイングレスを実行するための推奨される方法です。 FSM を使用すると、軽量プロファイルを維持しながら、さまざまなシナリオに対応する豊富なポリシー仕様を備えた高性能のイングレス コントローラーを利用できます。

FSM をイングレスとして使用するには、メッシュのインストール時に有効にします。

```bash
osm install --set fsm.enabled=true
```

適切な API を使用して FSM のエッジ プロキシを構成することに加えて、osm-edge のサービス メッシュ バックエンドは、承認されたエッジ プロキシまたはゲートウェイからのトラフィックのみを受け入れます。 osm-edge の [IngressBackend 仕様][1] により、クラスター管理者とアプリケーション所有者は、サービス メッシュ バックエンドがイングレス トラフィックを承認する方法を明示的に指定できます。 次のセクションでは、`IngressBackend` API と `HTTPProxy` API を組み合わせて使用して、HTTP および HTTPS イングレス トラフィックをメッシュ バックエンドにルーティングできるようにする方法について説明します。

入力トラフィックは常に許可されたクライアントに制限することをお勧めします。 これを行うには、osm-edge インストールが配置されている名前空間にある FSM エッジ プロキシのエンドポイントを監視するように osm-edge を有効にします。

```bash
kubectl label ns <osm namespace> openservicemesh.io/monitored-by=<mesh name>
```

#### FSM を使用した HTTP Ingress

最小限の [HTTPProxy][2] 構成と osm-edge の `IngressBackend`[1] 仕様は、イングレス トラフィックを名前空間 `test` のメッシュ サービス `foo` にルーティングするため、次のようになります。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fsm-ingress
  namespace: test
spec:
  ingressClassName: pipy
  rules:
  - host: foo-basic.bar.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: foo
            port:
              number: 80
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http # http implies no TLS
  sources:
    - kind: Service
      namespace: osm-system
      name: ingress-pipy-controller
```

上記の構成により、外部クライアントは `test` 名前空間で `foo` サービスにアクセスできます。

1. Ingress 構成は、`foo-basic.bar.com` の `Host:` ヘッダーを持つ外部ソースからの着信 HTTP トラフィックを `test` 名前空間のポート `80` の `foo` という名前のサービスにルーティングします。
2. IngressBackend は、osm-edge がインストールされている名前空間 (デフォルトは「osm-system」) の「ingress-pipy-controller」サービスという名前のエンドポイントのみが、「foo」サービスのポート「80」にアクセスできるように構成されています。 `test` 名前空間。

#### 例

osm-edge で FSM を使用してメッシュ サービスを外部に公開する方法の例については、[Ingress with FSM demo](docs/demos/ingress_fms) を参照してください。

### 2. 独自のイングレス コントローラーとゲートウェイを導入する

osm-edge を FSM でイングレスに使用することがユース ケースで実現できない場合、osm-edge は、独自のイングレス コントローラーとエッジ ゲートウェイを使用して、外部トラフィックをサービス メッシュ バックエンドにルーティングする機能を提供します。 上記の ingress の構成方法と同様に、サービス メッシュ バックエンドにトラフィックをルーティングするように ingress コントローラーを構成することに加えて、外部から発信されたトラフィックのプロキシを担当するクライアントを承認するには、IngressBackend 構成が必要です。

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.IngressBackendSpec