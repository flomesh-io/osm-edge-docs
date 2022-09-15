---
Title: 「osm-edge コントロール プレーン コンポーネント」
Description: 「コンポーネント」
Type: docs
Weight: 2
---

## osm-edge コンポーネントを調べる

一部の osm-edge コンポーネントは、デフォルトで選択された名前空間にインストールされます。デフォルトは `osm-system` です。 次の「kubectl」コマンドを使用して検査します。

```console
# Replace osm-system with the namespace where osm-edge is installed
$ kubectl get pods,svc,secrets,meshconfigs,serviceaccount --namespace osm-system
```

一部のクラスター全体の (名前空間ではない) osm-edge コンポーネントもインストールされます。 次の「kubectl」コマンドを使用して検査します。

```console
kubectl get clusterrolebinding,clusterrole,mutatingwebhookconfiguration
```

内部では、`osm` は [Helm](https://helm.sh) ライブラリを使用して、コントロール プレーンの名前空間に Helm の `release` オブジェクトを作成しています。 Helm の「リリース」名はメッシュ名です。 `helm` CLI を使用して、インストールされた Kubernetes マニフェストをより詳細に検査することもできます。 [Helm のインストール] 方法については、Helm のドキュメントを参照してください (https://helm.sh/docs/intro/install/)。

```console
$ helm get manifest osm --namespace <osm-namespace>
```

## コンポーネント

各コンポーネントを見てみましょう。

### (1) プロキシ コントロール プレーン

プロキシ コントロール プレーンは、[サービス メッシュ](https://www.bing.com/search?q=What%27s+a+service+mesh%3F) の運用において重要な役割を果たします。 すべてのプロキシは [サイドカー](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) としてインストールされ、プロキシ コントロール プレーンへの mTLS gRPC 接続を確立します。 プロキシは、構成の更新を継続的に受け取ります。 このコンポーネントは、選択された特定のリバース プロキシに必要なインターフェイスを実装します。 osm-edge は [Pipy Repo](https://flomesh.io/docs/en/operating/repo/0-intro) を実装しています。

### (2) 証明書マネージャー

Certificate Manager は、サービス メッシュに参加している各サービスに TLS 証明書を提供するコンポーネントです。
これらのサービス証明書は、mTLS を使用してサービス間の接続を確立および暗号化するために使用されます。

### (3) エンドポイント プロバイダー

エンドポイント プロバイダーは、サービス メッシュに参加しているコンピューティング プラットフォーム (Kubernetes クラスター、オンプレミス マシン、またはクラウド プロバイダーの VM) と通信する 1 つ以上のコンポーネントです。 エンドポイント プロバイダーは、サービス名を IP アドレスのリストに解決します。 エンドポイント プロバイダーは、仮想マシン、仮想マシン スケール セット、Kubernetes クラスターなど、それらが実装されているコンピューティング プロバイダーの特定のプリミティブを理解しています。

### (4) メッシュ仕様

メッシュ仕様は、既存の [SMI Spec](https://github.com/deislabs/smi-spec) コンポーネントのラッパーです。 このコンポーネントは、YAML 定義用に選択された特定のストレージを抽象化します。 このモジュールは事実上、[SMI Spec's Kubernetes informers](https://github.com/deislabs/smi-sdk-go) のラッパーであり、現在、ストレージ (Kubernetes/etcd) の仕様を抽象化しています。

### (5) メッシュカタログ

メッシュ カタログは osm-edge の中心的なコンポーネントであり、他のすべてのコンポーネントの出力を構造に結合し、プロキシ構成に変換して、プロキシ コントロール プレーンを介してすべてのリッスン プロキシにディスパッチできます。
このコンポーネント:

1. [mesh specification module (4)](#4-mesh-specification) と通信して、[Kubernetes service](https://kubernetes.io/docs/concepts/services-networking/service/) がいつ開始されたかを検出します。 [SMI Spec](https://github.com/deislabs/smi-spec) を使用して作成、変更、または削除します。
2. [certificate manager (2)](#2-certificate-manager) に連絡し、新しく検出されたサービスの新しい TLS 証明書を要求します。
3. [endpoints providers (3)](#3-endpoints-providers) を介してコンピューティング プラットフォームを監視することにより、メッシュ ワークロードの IP アドレスを取得します。
4. 上記の 1、2、および 3 の出力をデータ構造に結合し、[proxy control plane (1)](#1-proxy-control-plane) に渡し、シリアル化して、関連するすべての接続先に送信します。 プロキシ。

![diagram](https://user-images.githubusercontent.com/2224492/176060685-8504c433-c91b-4f9e-9754-f9ccb6c28a87.png)

### 6 サイドカー ドライバー

プロキシ コントロール プレーン コンポーネントは、サイドカー ドライバーのライフ サイクル管理と、**Sidecar Driver** インターフェイスを使用したドライバーとの通信を担当します。

サイドカーの拡張または提供を希望するベンダーは、抽象化を実装し、サイドカー固有のニーズに合った形式/方法で、それぞれのサイドカー実装に構成の更新を提供します。

サイドカー ドライバーは、OSM _Injector_ および _Controller_ と統合するために **Driver** インターフェイスを実装する必要があります。

![Sidecar Driver lifecycle](https://user-images.githubusercontent.com/95846930/175821540-9b7326ac-41e4-4f8e-b23d-bc6b0a5bb7c8.png)

## 詳細なコンポーネントの説明

このセクションでは、Open Service Mesh (osm-edge) の開発の指針となる規約について概説します。 このセクションで説明するコンポーネント:

  - (A) プロキシ [sidecar](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) - サービス メッシュ機能を備えた Pipy またはその他のリバース プロキシ
- (B) [Proxy Certificate](#b-proxy-tls-certificate) - [Certificate Manager](#2-certificate-manager) によって特定のプロキシに発行された一意の X.509 証明書
- (C) サービス - [Kubernetes service resource](https://kubernetes.io/docs/concepts/services-networking/service/) SMI 仕様で参照
- (D) [Service Certificate](#d-service-tls-certificate) - サービスに対して発行された X.509 証明書
- (E) ポリシー - [SMI Spec](https://smi-spec.io/) ターゲット サービスのプロキシによって適用されるトラフィック ポリシー
- 特定のサービスのトラフィックを処理するサービス エンドポイントの例:
  - (F) Azure VM - Azure VM で実行されているプロセスで、IP 1.2.3.11、ポート 81 で接続をリッスンします。
  - (G) Kubernetes Pod - Kubernetes クラスターで実行され、IP 1.2.3.12、ポート 81 で接続をリッスンするコンテナー。
  - (H) オンプレミス コンピューティング - お客様のプライベート データ センター内のマシンで実行され、IP 1.2.3.13、ポート 81 で接続をリッスンするプロセス。

[Service (C)](#c-service) には証明書 (D) が割り当てられ、SMI 仕様ポリシー (E) に関連付けられています。[Service (C)](#c-service) のトラフィックは、 各エンドポイントがプロキシ (A) で拡張されるエンドポイント (F、G、H)。プロキシ (A) には、サービス証明書 (D) とは異なる専用の証明書 (B) があり、mTLS 接続に使用されます。 プロキシから [proxy-control-plane](#1-proxy-control-plane) へ。
![service-relationships-diagram](https://user-images.githubusercontent.com/2224492/176343499-7b48094f-647e-421b-b349-03556fd0f90a.png)

### (C) サービス

上の図のサービスは、SMI 仕様で参照されている [Kubernetes service resource](https://kubernetes.io/docs/concepts/services-networking/service/) です。 例として、以下で定義され、「TrafficSplit」ポリシーによって参照される「bookstore」サービスがあります。


```yaml
apiVersion: v1
kind: Service
metadata:
  name: bookstore
  labels:
    app: bookstore
spec:
  ports:
    - port: 14001
      targetPort: 14001
      name: web-port
  selector:
    app: bookstore

---
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-traffic-split
spec:
  service: bookstore
  backends:
    - service: bookstore-v1
      weight: 100
```
### (A) プロキシ
  
  osm-edge では、`Proxy` は抽象的な論理コンポーネントとして定義されています。

- メッシュ サービス プロセス (Kubernetes または VM 上で実行されるコンテナーまたはバイナリー) の前に立つ
- プロキシ コントロール プレーン (xDS サーバー) への接続を維持します。
- [proxy control plane](#1-proxy-control-plane) (Pipy Repo 実装) から構成の更新 (xDS プロトコル バッファ) を継続的に受信します。
   osm-edge は、[Pipy](https://flomesh.io/) リバース プロキシの実装とともに出荷されます。

### (F、G、H) エンドポイント

osm-edge コードベース内で、「エンドポイント」は、コンテナまたは仮想マシンの IP アドレスとポート番号のタプルとして定義されます。プロキシをホストし、プロセスの前に立ち、サービスのメンバーであり、参加します。 サービスメッシュで。
[service endpoints (F,G,H)](#fgh-endpoint) は、[service (C)](#c-service) のトラフィックを処理する実際のバイナリです。
エンドポイントは、コンテナ、バイナリ、またはプロセスを一意に識別します。
IP アドレス、ポート番号を持ち、サービスに属します。
サービスは 0 個以上のエンドポイントを持つことができ、各エンドポイントは 1 つのサイドカー プロキシのみを持つことができます。 エンドポイントは単一のサービスに属している必要があるため、関連付けられたプロキシも単一のサービスに属している必要があります。

### (D) サービス TLS 証明書

特定のサービスを形成するエンドポイントの前にあるプロキシは、特定のサービスの証明書を共有します。
この証明書は、サービス メッシュ内の **他のサービス** のエンドポイントに面するピア プロキシとの mTLS 接続を確立するために使用されます。
サービス証明書は短命です。
各サービス証明書の有効期間は [約 24 時間](#certificate-lifetime) であり、証明書失効機能が不要になります。
osm-edge は、これらの証明書のタイプ `ServiceCertificate` を宣言します。
`ServiceCertificate` は、この種の証明書が開発者ドキュメントの [Interfaces](#interfaces) セクションでどのように参照されているかです。

### (E) ポリシー

上の図 (E) で参照されているポリシー コンポーネントは、[Service (C)](#c-
service）。 たとえば、サービス「bookstore」および「bookstore-v1」を参照する「TrafficSplit」:

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-traffic-split
spec:
  service: bookstore
  backends:
    - service: bookstore-v1
      weight: 100
```

### 証明書の有効期間

[Certificate Manager](#2-certificate-manager) によって発行されるサービス証明書は、有効期間が約 24 時間の短命の証明書です。
証明書の有効期限が短いため、明示的な失効メカニズムが不要になります。
特定の証明書の有効期限は、24 時間からランダムに短縮または延長されます。これは、基盤となる証明書管理システムに発生する [thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem) を回避するためです。 一方、プロキシ証明書は有効期間が長い証明書です。

### プロキシ証明書、プロキシ、およびエンドポイントの関係

- `ProxyCertificate` は、`Proxy` に対して osm-edge によって発行されます。これは、将来プロキシ コントロール プレーンに接続することが期待されます。証明書が発行された後、プロキシがプロキシ コントロール プレーンに接続する前は、証明書は「要求されていない」状態にあります。プロキシが証明書を使用してコントロール プレーンに接続すると、証明書の状態は「要求済み」に変わります。
- 「プロキシ」はリバース プロキシで、プロキシ コントロール プレーンへの接続を試みます。 「プロキシ」は、プロキシ コントロール プレーンへの接続を許可される場合と許可されない場合があります。
- `Endpoint` は `Proxy` によってフロントされ、`Service` のメンバーです。 osm-edge は、特定のサービスに属する [endpoints providers](#3-endpoints-providers) を介してエンドポイントを検出した可能性がありますが、osm-edge はプロキシを確認していません。これらのエンドポイントの前に立ち、プロキシ コントロール プレーンに接続します。まだ。

発行された `ProxyCertificates` ∩ 接続された `Proxy` ∩ 発見された `Endpoints` のセットの **交差** は、サービス メッシュの参加者のセットです。

![service-mesh-participants](https://user-images.githubusercontent.com/2224492/176342258-8f28b01e-8ef9-49e6-947b-544f9b2739fc.png)
