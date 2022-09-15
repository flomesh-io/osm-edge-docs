---
title: 「osm-edge コントロール プレーンをアップグレードする」
description: 「アップグレードガイド」
aliases: ["/docs/upgrade_guide","/docs/troubleshooting/cli/mesh_upgrade"]
type: docs
weight: 3
---

# アップグレードガイド

このガイドでは、osm-edge コントロール プレーンをアップグレードする方法について説明します。

## アップグレードの仕組み

osm-edge のコントロール プレーンのライフサイクルは Helm によって管理され、[Helm's upgrade functionality](https://helm.sh/docs/intro/using_helm/#helm-upgrade-and-helm-rollback-upgrading-a-release-and-recovering-on-failure)、変更された値とリソース テンプレートに基づいて、必要に応じてコントロール プレーン コンポーネントにパッチを適用または交換します。

### アップグレード中のリソースの可用性
アップグレードには新しいバージョンでの osm-controller の再デプロイが含まれる場合があるため、コントローラーのダウンタイムが発生する場合があります。 osm-controller が利用できない場合、新しい SMI リソースの処理に遅延が発生し、プロキシ サイドカー コンテナーを挿入する新しいポッドの作成が失敗し、mTLS 証明書がローテーションされません。

既存の SMI リソースは影響を受けません。これは、データ プレーン (Pipy サイドカー構成を含む) もアップグレードの影響を受けないことを意味します。

アップグレードに CRD の変更が含まれる場合、データ プレーンの中断が予想されます。 データ プレーンのアップグレードの合理化は、問題 [#512](https://github.com/openservicemesh/osm/issues/512) で追跡されています。

## ポリシー

特定のアップグレード パスのみがテストされ、サポートされています。

**注**: これらの計画は暫定的なものであり、変更される可能性があります。

このセクションの重大な変更は、次のユーザー向けコンポーネントに対する互換性のない変更を指します。
- `osm` CLI コマンド、フラグ、および動作
- SMI CRD およびコントローラ

これは、以下がユーザー向けではなく、非互換性がユーザー向けコンポーネントによって処理される限り、互換性のない変更は「破壊的」とは見なされないことを意味します。
- チャート値.yaml
- `osm-mesh-config` MeshConfig
- 内部で使用されるラベルと注釈 (監視者、インジェクション、指標など)

アップグレードは、以下で説明するように、重大な変更が含まれていないバージョン間でのみサポートされます。

osm-edge バージョン `0.y.z` の場合:
- `0.y.z` と `0.y.z+1` の間に破壊的変更は導入されません
- `0.y.z` と `0.y+1.0` の間に重大な変更が導入される可能性があります

`x >= 1` の osm-edge バージョン `x.y.z` の場合:
- `x.y.z` と `x.y+1.0` の間、または `x.y.z` と `x.y.z+1` の間に重大な変更は導入されません。
- `x.y.z` と `x+1.0.0` の間に重大な変更が導入される可能性があります

## osm-edge のアップグレード方法

メッシュをアップグレードする推奨される方法は、`osm` CLI を使用することです。 高度なユースケースでは、`helm` を使用できます。

### CRD のアップグレード
Helm は初期インストールを超えて CRD を管理しないため、osm-edge は「osm-bootstrap」ポッドの init-container を利用して、アップグレード中に既存の CRD を更新し、新しい CRD を追加します。 新しいリリースに既存の CRD の更新が含まれているか、新しい CRD が追加されている場合、「osm-bootstrap」ポッドの「init-osm-bootstrap」が CRD を更新します。 関連するカスタム リソースはそのまま維持されるため、アップグレードの前または直後に追加のアクションは必要ありません。

[リリース ノート](https://github.com/flomesh-io/osm-edge/releases) の「CRD 更新」セクションをチェックして、osm-edge で使用される CRD に更新が行われているかどうかを確認してください。 カスタム リソースのバージョンが、更新された CRD がサポートするバージョン内にある場合は、すぐに対応する必要はありません。 osm-edge はすべての CRD に変換 Webhook を実装し、古いバージョンのサポートを保証し、後でカスタム リソースを更新する柔軟性を提供します。

### osm-edge CLI によるアップグレード

**前提条件**

- osm-edge コントロール プレーンがインストールされた Kubernetes クラスター
     - Kubernetes クラスターに、新しい osm-edge チャートで必要な最小の Kubernetes バージョンがあることを確認します。 これは、[インストールの前提条件](/docs/getting_started/install#Pre-requisites) にあります。
- `osm` CLI をインストール
   - デフォルトでは、`osm` CLI は、インストールしたものと同じチャート バージョンにアップグレードします。 例えば `osm` CLI の v0.9.2 は、osm-edge Helm チャートの v0.9.2 にアップグレードされます。 CLI に一致するバージョン以外のバージョンの Helm チャートにアップグレードしても機能する可能性がありますが、これらのシナリオはテストされておらず、発生した問題が報告されても修正されない場合があります。

`osm mesh upgrade` コマンドは、メッシュの既存の Helm リリースの `helm upgrade` を実行します。

基本的な使用法では、追加の引数やフラグは必要ありません:
```console
$ osm mesh upgrade
osm-edge successfully upgraded mesh osm
```

このコマンドは、デフォルトの osm-edge 名前空間にあるデフォルトのメッシュ名でメッシュをアップグレードします。 デフォルトでは、以前のリリースの値は新しいリリースに引き継がれませんが、「osm mesh upgrade」の「--set」フラグを使用して個別に渡すことができます。

詳細については、「osm mesh upgrade --help」を参照してください

### Helm によるアップグレード

#### 前提条件

- osm-edge コントロール プレーンがインストールされた Kubernetes クラスター
- [helm 3 CLI](https://helm.sh/docs/intro/install/)

#### osm-edge 構成
アップグレードすると、osm-edge のインストールまたは実行に使用されたカスタム設定がデフォルトに戻される場合があります。これには、メトリックのデプロイのみが含まれます。 これらの値が上書きされないように、ガイドに注意深く従ってください。

osm-edge 構成に加えた変更を保持するには、`helm --values` フラグを使用します。 [値ファイル](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/values.yaml) のコピーを作成します (必ず使用してください アップグレードされたグラフのバージョン)、カスタマイズしたい値を変更します。 その他の値はすべて省略できます。

**注: MeshConfig に入る構成の変更は、アップグレード中に適用されず、値はアップグレード前のままになります。 MeshConfig の値を更新したい場合は、アップグレード後にリソースにパッチを適用することで実行できます。

たとえば、MeshConfig の「logLevel」フィールドがアップグレード前に「info」に設定されていた場合、これを「override.yaml」で更新しても、アップグレード中に変更は発生しません。

<b>警告:</b> `osm.meshName` または `osm.osmNamespace` を変更しないでください

#### 兜のアップグレード
次に、次の「helm upgrade」コマンドを実行します。
```console
$ helm upgrade <mesh name> osm --repo https://openservicemesh.github.io/osm --version <chart version> --namespace <osm namespace> --values override.yaml
```
デフォルト設定を使用する場合は、`--values` フラグを省略します。

その他のオプションについては、`helm upgrade --help` を実行してください。

## サードパーティの依存関係のアップグレード

### Pipy

Pipy のバージョンは、osm-mesh-config の `sidecarImage` 変数の値を変更することで更新できます。 たとえば、[Pipy image](https://hub.docker.com/r/flomesh/pipy) を最新に更新するには (これはあくまで例であり、最新のイメージは推奨されません)、次のコマンドを実行する必要があります。

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"sidecarImage": "flomesh/pipy:latest"}}}' --type=merge
```

MeshConfig リソースが更新された後、osm-edge によって実行される自動サイドカー インジェクションの一部として、更新されたバージョンの Pipy サイドカーが Pod にインジェクトされるように、メッシュの一部であるすべての Pod とデプロイを再起動する必要があります。 これは、「kubectl rollout restart deploy」コマンドで実行できます。

### プロメテウス、グラファーナ、イェーガー

有効にすると、osm-edge の Prometheus、Grafana、および Jaeger サービスが、他の osm-edge コントロール プレーン コンポーネントとともにデプロイされます。 これらのサード パーティの依存関係は、Pipy のように meshconfig を介して更新することはできませんが、バージョンはデプロイで直接更新できます。 たとえば、prometheus を v2.19.1 に更新するには、ユーザーは次のコマンドを実行できます。

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
kubectl set image deployment/osm-prometheus -n $osm_namespace prometheus="prom/prometheus:v2.19.1"
```

Grafana 8.1.0 に更新するには、コマンドは次のようになります。

```bash
kubectl set image deployment/osm-grafana -n $osm_namespace grafana="grafana/grafana:8.1.0"
```

Jaeger の場合、ユーザーは次のコマンドを実行して 1.26.0 に更新します。

```bash
kubectl set image deployment/jaeger -n $osm_namespace jaeger="jaegertracing/all-in-one:1.26.0"
```

## osm-edge アップグレード トラブルシューティング ガイド

#### osm-edge メッシュ アップグレードのタイムアウト

### 不十分な CPU

「osm mesh upgrade」コマンドがタイムアウトする場合は、CPU が不足している可能性があります。
1. ポッドをチェックして、完全に稼働していないポッドがないかどうかを確認します
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl get pods -n osm-system
```
2. Pending 状態の Pod がある場合は、「kubectl describe」を使用して「Events」セクションを確認します。
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl describe pod <pod-name> -n osm-system
```

次のエラーが表示された場合は、Docker が使用できる CPU の数を増やしてください。
```bash
`Warning  FailedScheduling  4s (x15 over 19m)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.`
```
#### CLI パラメータの検証エラー

「osm mesh upgrade」コマンドがまだタイムアウトする場合は、CLI/イメージ バージョンの不一致が原因である可能性があります。

1. ポッドをチェックして、完全に稼働していないポッドがないかどうかを確認します
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl get pods -n osm-system
```
2. Pending 状態の Pod がある場合は、「kubectl describe」を使用して、「Error Validating CLI parameters」の「Events」セクションを確認します。
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl describe pod <pod-name> -n osm-system
```
3. エラーが見つかった場合は、ポッドのログでエラーを確認してください
```bash
kubectl logs -n osm-system <pod-name> | grep -i error
```

次のエラーが表示される場合は、CLI とイメージ バージョンの不一致が原因です。
```bash
`"error":"Please specify the init container image using --init-container-image","reason":"FatalInvalidCLIParameters"`
```
回避策は、「osm mesh upgrade」の実行時に「container-registry」および「osm-image-tag」フラグを設定することです。
```bash
osm mesh upgrade --container-registry $CTR_REGISTRY --osm-image-tag $CTR_TAG --enable-egress=true
```

### その他の問題 

上記の手順で解決されない問題が発生している場合は、[open a GitHub issue](https://github.com/flomesh-io/osm-edge/issues) を行ってください。
