---
title: 「iptables リダイレクト」
description: 「iptables リダイレクト」
type: docs
weight: 1
---

# iptables リダイレクト

osm-edge は [iptables](https://linux.die.net/man/8/iptables) を活用して、サービス メッシュに参加しているポッドとの間のトラフィックを傍受し、各ポッドで実行されている Pipy プロキシ サイドカー コンテナーにリダイレクトします。 Pipy プロキシ サイドカーにリダイレクトされたトラフィックは、サービス メッシュ トラフィック ポリシーに基づいてフィルタリングおよびルーティングされます。

## 使い方

osm-edge サイドカー インジェクター サービス「osm-injector」は、サービス メッシュ内で作成されたすべてのポッドに Pipy プロキシ サイドカーを挿入します。 Pipy プロキシ サイドカーに加えて、`osm-injector` は [init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) も注入します。これは、アプリケーションの前に実行される特殊なコンテナーです。 ポッド内のコンテナー。 注入された init コンテナーは、ポッドからのすべての送信 TCP トラフィックとポッドへのすべての受信トラフィック TCP トラフィックがそのポッドで実行されている pipy プロキシ サイドカーにリダイレクトされるように、トラフィック リダイレクト ルールを使用してアプリケーション ポッドをブートストラップする役割を果たします。 このリダイレクトは、一連の「iptables」コマンドを実行することにより、init コンテナーによって設定されます。

### トラフィックのリダイレクト用に予約されたポート

osm-edge は、一連のポート番号を予約して、トラフィックのリダイレクトを実行し、Pipy プロキシ サイドカーへの管理者アクセスを提供します。 これらのポート番号は、メッシュで実行されているアプリケーション コンテナーで使用してはならないことに注意してください。 これらの予約済みポート番号のいずれかを使用すると、Pipy プロキシ サイドカーが正しく機能しなくなります。

以下は、osm-edge で使用するために予約されているポート番号です:

1. `15000`: 現在の構成ファイルを返すために、`localhost` で公開された Pipy 管理インターフェイスによって使用されます。
2. `15001`: ポッド内のアプリケーションによって送信されたアウトバウンド トラフィックを受け入れてプロキシするために、Pipy アウトバウンド リスナーによって使用されます。
3. `15003`: Pipy インバウンド リスナーによって使用され、ポッド内のアプリケーション宛てのポッドに入るインバウンド トラフィックを受け入れてプロキシします。
4. `15010`: Pipy インバウンド Prometheus リスナーによって使用され、Pipy の Prometheus メトリクスのスクレイピングに関連するインバウンド トラフィックを受け入れ、プロキシします。
5. `15901`: 書き換えられた HTTP liveness プローブを提供するために Pipy によって使用されます
6. `15902`: 書き換えられた HTTP 準備プローブを提供するために Pipy によって使用されます
7. `15903`: 書き換えられた HTTP スタートアップ プローブを提供するために Pipy によって使用されます

以下は、osm-edge で使用するために予約されており、トラフィックが Pipy をバイパスできるようにするポート番号です。
1. `15904`: `osm-healthcheck` によって使用され、`httpGet` ヘルス プローブに書き換えられた `tcpSocket` ヘルス プローブを提供します。

### トラフィックのリダイレクト用に予約されたアプリケーション ユーザー ID (UID)

osm-edge は、Pipy プロキシ サイドカー コンテナー用にユーザー ID (UID) 値「1500」を予約します。 このユーザー ID は、トラフィックのインターセプトとリダイレクトを実行してリダイレクトがループにならないようにする際に最も重要です。 ユーザー ID 値「1500」は、リダイレクト ルールをプログラムするために使用され、Pipy からリダイレクトされたトラフィックがそれ自体にリダイレクトされないようにします。

アプリケーション コンテナは、予約済みのユーザー ID 値「1500」を使用してはなりません。

### 傍受されたトラフィックの種類

現在、osm-edge は各ポッドの Pipy プロキシ サイドカーをプログラムして、インバウンドおよびアウトバウンドの「TCP」トラフィックのみをインターセプトします。 これには、生の「TCP」トラフィックと、「HTTP」、「gRPC」など、基礎となるトランスポート プロトコルとして「TCP」を使用するアプリケーション トラフィックが含まれます。これは、「iptables」によって傍受される可能性のある「UDP」および「ICMP」トラフィックを意味します。 ` は傍受されず、Pipy プロキシ サイドカーにリダイレクトされます。

### iptables チェーンとルール

osm-edge の「osm-injector」サービスは、init コンテナーをプログラムして、一連の「iptables」チェーンとルールを設定し、トラフィックの傍受とリダイレクトを実行します。 次のセクションでは、これらのチェーンとルールの役割について詳しく説明します。

osm-edge は 4 つのチェーンを活用して、トラフィックの傍受とリダイレクトを実行します。

1. `PROXY_INBOUND`: ポッドに入るインバウンド トラフィックをインターセプトするためのチェーン
1. `PROXY_IN_REDIRECT`: インターセプトされたインバウンド トラフィックをサイドカー プロキシのインバウンド リスナーにリダイレクトするチェーン
1. `PROXY_OUTPUT`: ポッド内のアプリケーションからのアウトバウンド トラフィックをインターセプトするためのチェーン
1. `PROXY_REDIRECT`: インターセプトされたアウトバウンド トラフィックをサイドカー プロキシのアウトバウンド リスナーにリダイレクトするチェーン

上記の各チェーンには、Pipy プロキシ サイドカーを介してアプリケーション トラフィックをインターセプトおよびリダイレクトするルールがプログラムされています。

## 送信 IP 範囲の除外

アプリケーションからのアウトバウンド TCP ベースのトラフィックは、デフォルトで、osm-edge によってプログラムされた「iptables」ルールを使用して傍受され、Pipy プロキシ サイドカーにリダイレクトされます。 場合によっては、サービス メッシュ ポリシーに基づいて、特定の IP 範囲が Pipy プロキシ サイドカーによってリダイレクトおよびルーティングされないようにすることが望ましい場合があります。 IP 範囲を除外する一般的な使用例は、Kubernetes API サーバー宛てのトラフィックやクラウド プロバイダーのインスタンス メタデータ サービス宛てのトラフィックなど、Pipy プロキシ経由で非アプリケーション ロジック ベースのトラフィックをルーティングしないことです。 このようなシナリオでは、特定の IP 範囲をサービス メッシュ トラフィック ルーティング ポリシーの対象から除外することが必要になります。

アウトバウンド IP 範囲は、グローバル メッシュ スコープまたはポッド スコープごとに除外できます。

### 1. グローバル アウトバウンド IP 範囲の除外

osm-edge は、次のように、メッシュ内のすべてのポッドに適用されるアウトバウンド トラフィック インターセプトから除外する IP 範囲のグローバル リストを指定する手段を提供します。

1. `--set` オプションを使用した osm-edge インストール中:
    ```bash
    # To exclude the IP ranges 1.1.1.1/32 and 2.2.2.2/24 from outbound interception
    osm install --set="osm.outboundIPRangeExclusionList={1.1.1.1/32,2.2.2.2/24}
    ```

1. `osm-mesh-config` リソースの `outboundIPRangeExclusionList` フィールドを設定することにより:
    ```bash
    ## Assumes osm-edge is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["1.1.1.1/32", "2.2.2.2/24"]}}}'  --type=merge
    ```

   インストール後の除外対象として IP 範囲が設定されている場合は、この変更を有効にするために、監視対象の名前空間でポッドを再起動してください。

グローバルに除外された IP 範囲は、`osm-mesh-config` `MeshConfig` カスタム リソースに保存され、`osm-injector` によるサイドカー インジェクション時に読み取られます。 これらの動的に構成可能な IP 範囲は、Pipy プロキシ サイドカーを介してトラフィックをインターセプトおよびリダイレクトするために使用される静的ルールと共に、init コンテナーによってプログラムされます。 除外された IP 範囲は、Pipy プロキシ サイドカーへのトラフィック リダイレクトのために傍受されません。 詳細については、[outbound IP range exclusion demo](docs/demos/outbound_ip_exclusion) を参照してください。

### 2. ポッド スコープの送信 IP 範囲の除外

IP CIDR 範囲のカンマ区切りリストを「openservicemesh.io/outbound-ip-range-exclusion-list=<IP CIDR のカンマ区切りリスト>」として指定するように Pod にアノテーションを付けることで、送信 IP 範囲の除外を Pod スコープで構成できます。

```bash
# To exclude the IP ranges 10.244.0.0/16 and 10.96.0.0/16 from outbound interception on the pod
kubectl annotate pod <pod> openservicemesh.io/outbound-ip-range-exclusion-list="10.244.0.0/16,10.96.0.0/16"
```

Pod の作成後に IP 範囲にアノテーションが付けられている場合は、対応する Pod を再起動してこの変更を有効にしてください。

## 送信 IP 範囲の包含

アプリケーションからのアウトバウンド TCP ベースのトラフィックは、デフォルトで、osm-edge によってプログラムされた「iptables」ルールを使用して傍受され、Pipy プロキシ サイドカーにリダイレクトされます。 場合によっては、サービス メッシュ ポリシーに基づいて特定の IP 範囲のみを Pipy プロキシ サイドカーによってリダイレクトおよびルーティングし、残りのトラフィックをサイドカーにプロキシしないことが望ましい場合があります。 このようなシナリオでは、包含 IP 範囲を指定できます。

アウトバウンドの包含 IP 範囲は、グローバル メッシュ スコープまたはポッド スコープごとに指定できます。

### 1. グローバル アウトバウンド IP 範囲の包含

osm-edge は、次のように、メッシュ内のすべてのポッドに適用可能なアウトバウンド トラフィック インターセプトに含める IP 範囲のグローバル リストを指定する手段を提供します。

1. `--set` オプションを使用した osm-edge インストール中:
    ```bash
    # To include the IP ranges 1.1.1.1/32 and 2.2.2.2/24 for outbound interception
    osm install --set="osm.outboundIPRangeInclusionList={1.1.1.1/32,2.2.2.2/24}
    ```

1. `osm-mesh-config` リソースの `outboundIPRangeInclusionList` フィールドを設定することにより:
    ```bash
    ## Assumes osm-edge is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundIPRangeInclusionList":["1.1.1.1/32", "2.2.2.2/24"]}}}'  --type=merge
    ```

   インストール後に含める IP 範囲が設定されている場合は、この変更を有効にするために、監視対象の名前空間でポッドを再起動してください。

グローバルに含まれる IP 範囲は、`osm-mesh-config` `MeshConfig` カスタム リソースに保存され、`osm-injector` によるサイドカー インジェクション時に読み取られます。 これらの動的に構成可能な IP 範囲は、Pipy プロキシ サイドカーを介してトラフィックをインターセプトおよびリダイレクトするために使用される静的ルールと共に、init コンテナーによってプログラムされます。 指定された包含 IP 範囲外の IP アドレスは、トラフィックを Pipy プロキシ サイドカーにリダイレクトするために傍受されません。

### 2. Pod スコープのアウトバウンド IP 範囲の包含

IP CIDR 範囲のコンマ区切りリストを `openservicemesh.io/outbound-ip-range-inclusion-list=<IP CIDR のカンマ区切りリスト>` として指定するようにポッドにアノテーションを付けることにより、送信 IP 範囲の包含をポッド スコープで構成できます。

```bash
# To include the IP ranges 10.244.0.0/16 and 10.96.0.0/16 for outbound interception on the pod
kubectl annotate pod <pod> openservicemesh.io/outbound-ip-range-inclusion-list="10.244.0.0/16,10.96.0.0/16"
```

Pod の作成後に IP 範囲にアノテーションが付けられている場合は、対応する Pod を再起動してこの変更を有効にしてください。

## 送信ポートの除外

アプリケーションからのアウトバウンド TCP ベースのトラフィックは、デフォルトで、osm-edge によってプログラムされた「iptables」ルールを使用して傍受され、Pipy プロキシ サイドカーにリダイレクトされます。 場合によっては、サービス メッシュ ポリシーに基づいて、Pipy プロキシ サイドカーによって特定のポートをリダイレクトおよびルーティングしないことが望ましい場合があります。 ポートを除外する一般的な使用例は、コントロール プレーン トラフィックなど、Pipy プロキシ経由で非アプリケーション ロジック ベースのトラフィックをルーティングしないことです。 このようなシナリオでは、特定のポートをサービス メッシュ トラフィック ルーティング ポリシーの対象から除外することが必要になります。

アウトバウンド ポートは、グローバル メッシュ スコープまたはポッド スコープごとに除外できます。

### 1. グローバルな送信ポートの除外

osm-edge は、次のように、メッシュ内のすべてのポッドに適用されるアウトバウンド トラフィック インターセプトから除外するポートのグローバル リストを指定する手段を提供します。

1. `--set` オプションを使用した osm-edge インストール中:
    ```bash
    # To exclude the ports 6379 and 7070 from outbound sidecar interception
    osm install --set="osm.outboundPortExclusionList={6379,7070}
    ```

1. `osm-mesh-config` リソースの `outboundPortExclusionList` フィールドを設定することにより:
    ```bash
    ## Assumes osm-edge is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundPortExclusionList":[6379, 7070]}}}'  --type=merge
    ```

   インストール後にポートを除外するように設定した場合は、監視対象の名前空間でポッドを再起動して、この変更を有効にしてください。

グローバルに除外されたポートは、`osm-mesh-config` `MeshConfig` カスタム リソースに保存され、`osm-injector` によるサイドカー インジェクション時に読み取られます。 これらの動的に構成可能なポートは、Pipy プロキシ サイドカーを介してトラフィックをインターセプトおよびリダイレクトするために使用される静的ルールと共に、init コンテナーによってプログラムされます。 除外されたポートは、Pipy プロキシ サイドカーへのトラフィック リダイレクトのために傍受されません。

### 2. Pod スコープの送信ポートの除外

送信ポートの除外は、Pod スコープで構成できます。そのためには、「openservicemesh.io/outbound-port-exclusion-list=<comma Separated list of ports>」のようにポートのコンマ区切りリストで Pod に注釈を付けます。

```bash
# To exclude the ports 6379 and 7070 from outbound interception on the pod
kubectl annotate pod <pod> openservicemesh.io/outbound-port-exclusion-list=6379,7070
```

Pod の作成後にポートにアノテーションが付けられている場合は、この変更を有効にするために対応する Pod を再起動してください。

## 受信ポートの除外

上記のアウトバウンド ポートの除外と同様に、ポッドのインバウンド トラフィックは、トラフィックが送信されるポートに基づいて、サイドカーへのプロキシから除外できます。

### 1. グローバル受信ポートの除外

osm-edge は、次のように、メッシュ内のすべてのポッドに適用されるインバウンド トラフィックの傍受から除外するポートのグローバル リストを指定する手段を提供します。

1. `--set` オプションを使用した osm-edge インストール中:
    ```bash
    # To exclude the ports 6379 and 7070 from inbound sidecar interception
    osm install --set="osm.inboundPortExclusionList={6379,7070}
    ```

1. `osm-mesh-config` リソースの `inboundPortExclusionList` フィールドを設定することにより:
    ```bash
    ## Assumes osm-edge is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"inboundPortExclusionList":[6379, 7070]}}}'  --type=merge
    ```

   インストール後にポートを除外するように設定した場合は、監視対象の名前空間でポッドを再起動して、この変更を有効にしてください。

### 2. Pod スコープのインバウンド ポートの除外

「openservicemesh.io/inbound-port-exclusion-list=<カンマ区切りのポートのリスト>」のように、ポートのコンマ区切りリストで Pod に注釈を付けることで、インバウンド ポートの除外を Pod スコープで構成できます。

```bash
# To exclude the ports 6379 and 7070 from inbound sidecar interception on the pod
kubectl annotate pod <pod> openservicemesh.io/inbound-port-exclusion-list=6379,7070
```

Pod の作成後にポートにアノテーションが付けられている場合は、この変更を有効にするために対応する Pod を再起動してください。