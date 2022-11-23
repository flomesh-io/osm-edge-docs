---
title: 「名前空間追加」
description: "このセクションでは、osm-edge が Kubernetes 名前空間を監視する方法と理由について説明します"
Alias: "/docs/tasks/namespace_monitoring"
type: docs
weight: 2
---

# 名前空間の追加

## 概要

osm-edge コントロール プレーン (「メッシュ」とも呼ばれます) をセットアップする場合、一連の Kubernetes 名前空間をメッシュに登録することもできます。 名前空間を osm-edge に登録すると、osm-edge はその名前空間内のリソースを監視できるようになります。これは、それらが Pod、サービス、または SMI リソースとして表されるトラフィック ポリシーに展開されたアプリケーションであるかどうかにかかわらずです。

名前空間を監視できるメッシュは 1 つだけなので、同じ Kubernetes クラスター内に osm-edge の複数のインスタンスがある場合は注意が必要です。 ポリシーをアプリケーションに適用する場合、osm-edge は監視対象の名前空間のリソースのみを評価するため、アプリケーションが展開される名前空間を正しいメッシュ名を持つ osm-edge の正しいインスタンスに登録することが重要です。
名前空間を登録すると、オプションで、指定された名前空間内のリソースのメトリクスを収集したり、名前空間内のポッドにサイドカー プロキシ コンテナーを自動的に挿入したりできます。 これらはすべて、osm-edge がトラフィック管理とオブザーバビリティの機能を提供するのに役立つ機能です。 この機能を名前空間レベルでスコープすることで、チームはどのセグメントの
それらのクラスターは、どのメッシュの一部である必要があります。

名前空間の監視、自動サイドカー インジェクション、メトリック コレクションは、特定のラベルと注釈を Kubernetes 名前空間に追加することによって制御されます。 これは手動で、または「osm」CLI を使用して行うことができますが、「osm」CLI を使用することが推奨される方法です。 ラベル「openservicemesh.io/monitored-by=<mesh-name>」の存在により、指定された「mesh-name」を持つ osm-edge コントロール プレーンが監視できるようになります。
その名前空間内のすべてのリソース。 注釈 `openservicemesh.io/sidecar-injection=enabled` により、osm-edge は、その名前空間内で作成されたすべての Pod にサイドカー プロキシ コンテナを自動的に注入できます。 メトリクス アノテーション `openservicemesh.io/metrics=enabled` により、osm-edge は名前空間内のリソースに関するメトリクスを収集できます。

osm-edge CLI を使用して名前空間の監視を管理する方法については、以下を参照してください。

## osm-edge コントロール プレーンに名前空間を追加する

次のコマンドを使用して、監視とサイドカー インジェクション用の名前空間をメッシュに追加します。

```bash
osm namespace add <namespace>
```

[こちら](/docs/guides/app_onboarding/sidecar_injection/#explicitly-disabling-automatic-sidecar-injection-on-namespaces) に示すように、`--disable-sidecar-injection` フラグを使用して名前空間を追加する際にサイドカー インジェクションを明示的に無効にします。

## osm-edge コントロール プレーンから名前空間を削除します

次のコマンドを使用して、メッシュによる監視対象から名前空間を削除し、サイドカー インジェクションを無効にします。

```bash
osm namespace remove <namespace>
```

このコマンドは、名前空間の osm-edge 固有のラベルと注釈を削除し、メッシュから削除します。

## 名前空間のメトリックを有効にする

```bash
osm metrics enable --namespace <namespace>
```

## 名前空間を無視する

クラスターには、メッシュの一部であってはならない名前空間が存在する場合があります。 osm-edge から名前空間を明示的に除外するには:

```bash
osm namespace ignore <namespace>
```

## メッシュの名前空間の一部を一覧表示する

特定のメッシュ内の名前空間を一覧表示するには:

```bash
osm namespace list --mesh-name=<mesh-name>
```

## トラブルシューティングガイド

### ポリシーの問題

名前空間内のリソースに適用されている SMI ポリシーに変更が見られない場合は、名前空間が正しいメッシュに登録されていることを確認してください。

```bash
osm namespace list --mesh-name=<mesh-name>

NAMESPACE         MESH   SIDECAR-INJECTION
<namespace>       osm    enabled
```

名前空間が表示されない場合は、「kubectl」を使用して名前空間のラベルを確認してください。

```bash
kubectl get namespace <namespace> --show-labels

NAME          STATUS   AGE   LABELS
<namespace>   Active   36s   openservicemesh.io/monitored-by=<mesh-name>
```

ラベルの値が予期された `mesh-name` でない場合は、メッシュから名前空間を削除し、正しい `mesh-name` を使用して追加し直してください。

```bash
osm namespace remove <namespace> --mesh-name=<current-mesh-name>
osm namespace add <namespace> --mesh-name=<expected-mesh-name>
```

監視対象ラベルが存在しない場合は、メッシュに追加されていないか、メッシュへの追加時にエラーが発生しました。
`osm` CLI または kubectl を使用して名前空間をメッシュに追加します。

```bash
osm namespace add <namespace> --mesh-name=<mesh-name>
```

```bash
kubectl label namespace <namespace> openservicemesh.io/monitored-by=<mesh-name>
```

### 自動サイドカー インジェクションに関する問題

Pod にサイドカー コンテナーが自動的に挿入されていない場合は、サイドカー インジェクションが有効になっていることを確認します。

```bash
osm namespace list --mesh-name=<mesh-name>

NAMESPACE         MESH   SIDECAR-INJECTION
<namespace>       osm    enabled
```

名前空間が表示されない場合は、「kubectl」を使用して名前空間の注釈を確認してください。

```bash
kubectl get namespace <namespace> -o=jsonpath='{.metadata.annotations.openservicemesh\.io\/sidecar-injection}'
```

出力が「有効」以外の場合は、「osm」CLI を使用して名前空間を追加するか、「kubectl」で注釈を追加します。

```bash
osm namespace add <namespace> --mesh-name=<mesh-name> --disable-sidecar-injection=false
```

```bash
kubectl annotate namespace <namespace> openservicemesh.io/sidecar-injection=enabled --overwrite
```

### 指標収集に関する問題

特定の名前空間のリソースのメトリックが表示されない場合は、メトリックが有効になっていることを確認してください。

```bash
kubectl get namespace <namespace> -o=jsonpath='{.metadata.annotations.openservicemesh\.io\/metrics}'
```

出力が「有効」以外の場合は、「osm」CLI を使用して名前空間を有効にするか、「kubectl」で注釈を追加します。

```bash
osm metrics enable --namespace <namespace>
```

```bash
kubectl annotate namespace <namespace> openservicemesh.io/metrics=enabled --overwrite
```

### その他の問題

上記のデバッグ手法で解決されていない問題が発生している場合は、リポジトリで GitHub の問題を開いてください。
