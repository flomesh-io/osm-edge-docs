---
title: 「HA 設計の考慮事項」
description: 「osm-edge HA 設計の考慮事項」
Alias: "/docs/HA/"
type: docs
---

# 高可用性

osm-edge のコントロール プレーン コンポーネントは、高可用性と耐障害性を念頭に置いて構築されています。 次のセクションでは、これらの問題にどのように対処するかを詳しく説明します。

## 高可用性と耐障害性

高可用性と耐障害性は、osm-edge のいくつかの設計決定と外部メカニズムによって実装および保証されます。これについては、次の点で文書化されます。

### 無国籍

osm-edge のコントロール プレーン コンポーネントは、実行時に保存する必要がある状態依存データを所有していません。 以下の制御された例外を除いて:

- CA / ルート証明書: CA ルート証明書は、複数のレプリカを実行する場合、複数の osm-edge インスタンスで同じである必要があります。[証明書マネージャー](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/DESIGN.md#2-certificate-manager) の実装では、ルート CA が生成されている必要があります/ 以前の osm-edge 実行 (Vault、Cert-Manager) が提供されている場合、ルート CA は、すべてのインスタンスによる起動時にプロバイダーから取得されます。
   CA が存在しない場合に CA を自動生成できる他の証明書プロバイダー (Tresor など) の場合、作成中に原子性と同期が保証され、すべてのインスタンスが同じ CA をロードすることが保証されます。

これらの例外を除いて、残りの構成はビルドされ、Kubernetes から取得されます。

トラフィック ポリシーの計算に使用されるドメインの状態は、さまざまなランタイム プロバイダー (Kubernetes など) と、「osm-controller」がサブスクライブする関連オブジェクトの Kubernetes クライアント ゴー インフォーマーによって完全に提供されます。

実行中の複数の「osm-controller」は、同じオブジェクト セットをサブスクライブし、サービス メッシュに対して同一の構成を生成します。 client-go Kubernetes インフォーマーの性質により、最終的に一貫性があるため、「osm-controller」はポリシーの適用が最終的に一貫性があることを保証します。

<p align="center">
  <img src="/docs/images/ha/ha1.png" width="400" height="350"/>
</p>

### 再始動性

以前のステートレス設計の考慮事項により、osm-edge のコントロール プレーン コンポーネントが完全に再起動可能であることを保証する必要があります。

- インスタンスを再起動すると、すべての Kubernetes ドメイン リソースが再同期されます。 既存のプロキシは再接続され、(メッシュ トポロジまたはポリシーで変更が発生していないと仮定して) 同じ構成が再計算され、新しいバージョンとしてプロキシにプッシュされる必要があります。

### 水平スケーリング

コンポーネント `osm-controller` と `osm-injector` は、負荷または可用性の要件に応じて、別々の水平方向のスケーリングを可能にします。

- `osm-controller` が複数のレプリカで生成されると、接続プロキシは負荷分散され、コントロール プレーンで実行されている既存の osm-edge インスタンスのいずれかに接続されます。
- 同様に、`osm-injector` を水平方向にスケーリングして、メッシュ上のポッド オンボーディングの数/速度の増加を処理できます。
- `osm-controller` では、サービス証明書 (プロキシ間で TLS を認証し、相互に通信するために使用) は短命であり、コントロール プレーンの実行時にのみ保持されます (ただし、必要に応じてプロキシ xDS プロトコルの一部としてプッシュされます)。

  複数の「osm-controller」インスタンスは、単一のサービスに対して異なるが有効なサービス証明書を作成する場合があります。 これらの異なる証明書は、(1) 複数の osm-edge インスタンスが同じルート CA をロードする必要があるため、同じルートによって署名されており、(2) 照合に使用される同じ共通名 (CN) を持っています。 サービス間でトラフィックがプロキシされるときに認証します。

<p align="center">
  <img src="/docs/images/ha/ha2.png" width="450" height="400"/>
</p>

つまり、プロキシが接続するコントロール プレーンに関係なく、正しい/適切な CN を持ち、共有コントロール プレーンのルート CA によって署名された有効な証明書がプロキシにプッシュされます。

- 水平スケールを大きくしても、接続が切断されていない限り、確立されたプロキシ接続がコントロール プレーンに再配布されません。
- 水平スケールを小さくすると、切断されたプロキシが、ダウンスケールによって終了されなかったインスタンスに接続されます。 構成の新しいバージョンを計算し、接続を新たに確立するときにプッシュする必要があります。

<p align="center">
  <img src="/docs/images/ha/ha3.png" width="450" height="400"/>
</p>

- コントロール プレーンが完全にダウンした場合、実行中のプロキシは、実行中のコントロール プレーンに再接続できるようになるまで、ヘッドレス<sup>[1]</sup> モードで動作し続ける必要があります。

[1] ヘッドレス: 通常、コントロール プレーン/データ プレーンの設計パラダイムで言及されます。2 つのコンポーネント間に依存関係がある場合、依存エージェントが生き残り、依存エージェントが停止するか到達不能になったときに最新の状態で実行し続けることを可能にする概念を指します。

### 水平ポッド自動スケーリング - HPA

HPA は、平均ターゲット CPU 使用率 (%) と平均ターゲット メモリ使用率 (%) に基づいて、コントロール プレーン ポッドを自動的にスケールアップまたはスケールダウンします。
HPA を有効にするには、次のコマンドを使用します。

```bash
osm install --set=osm.<control_plane_pod>.autoScale.enable=true
```

> 注: control_plane_pod は「osmController」または「injector」にすることができます

コントロール プレーン ポッドの展開に適用されるポッドのアンチアフィニティにより、ノード間のポッドの分散の回復力が向上します。
ポッドの複数のレプリカが存在する場合、可能であれば異なるノードでスケジュールされます。

HPA の追加パラメーター:

- `minReplicas` (整数): オートスケーラーが設定できるポッドの最小数 (許容値: 1-10)
- `maxReplicas` (整数): オートスケーラーが設定できるポッドの最大数 (許容値: 1-10)
- `cpu.targetAverageUtilization` (int): パーセンテージで表される CPU 使用率の目標値 (許容値: 0-100)
- `memory.targetAverageUtilization` (int): パーセンテージで表されるメモリ使用率の目標値 (許容値: 0-100)

### ポッド中断予算 - PDB

計画停止中の中断を防ぐために、コントロール プレーン ポッド「osm-controller」と「osm-injector」には、各コントロール プレーン アプリケーションに対応する少なくとも 1 つのポッドが常に存在することを保証する PDB があります。

PDB を有効にするには、次のコマンドを使用します。

```bash
osm install --set=osm.<control_plane_pod>.enablePodDisruptionBudget=true
```

> 注: control_plane_pod は「osmController」または「injector」にすることができます
