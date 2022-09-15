---
title: 「調停者ガイド」
description: 「調停者ガイド」
aliases: ["/docs/reconciler_guide"]
type: docs
weight: 8
---

# 調停者ガイド

このガイドでは、osm-edge でリコンサイラーを有効にする方法について説明します。

## 調停者の仕組み

osm-edge でリコンサイラーを構築する目的は、osm-edge のコントロール プレーンの正しい操作に必要なリソースが常に望ましい状態になるようにすることです。 osm-edge インストールの一部としてインストールされ、「openservicemesh.io/reconcile: true」および「app.kubernetes.io/name: openservicemesh.io」というラベルを持つリソースは、調整ツールによって調整されます。

**注**: 調整可能なリソースでラベル `openservicemesh.io/reconcile: true` および `app.kubernetes.io/name: openservicemesh.io` が変更または削除された場合、調整ツールは期待どおりに動作しません。

調整可能なリソースの更新または削除イベントは、調整ツールをトリガーし、リソースを目的の状態に調整します。 調整可能なリソースでは、メタデータの変更 (名前の変更を除く) のみが許可されます。

### 調整されたリソース

osm-edge が調整するリソースは次のとおりです。

- CRD : osm-edge によってインストール/要求される CRD [CRDs for osm-edge](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/cmd/osm -bootstrap/crds) が調整されます。 osm-edge は必要な CRD のインストールとアップグレードを管理するため、osm-edge はそれらを調整して、それらの仕様、保存および提供されたバージョンが常に osm-edge によって必要とされる状態であることを保証します。

- MutatingWebhookConfiguration : MutatingWebhookConfiguration は、osm-edge のコントロール プレーンの一部としてデプロイされ、自動サイドカー インジェクションを有効にします。 これはポッドがメッシュに参加するための非常に重要なコンポーネントであるため、osm-edge はこのリソースを調整します。

- ValidatingWebhookConfiguration : ValidatingWebhookConfiguration は、さまざまなメッシュ構成を検証するために、osm-edge のコントロール プレーンの一部としてデプロイされます。 このリソースはメッシュに適用される構成を検証するため、osm-edge はこのリソースを調整します。


## reconciler で osm-edge をインストールする方法

reconciler を使用して osm-edge をインストールするには、次のコマンドを使用します。

```console
$ osm install --set osm.enableReconciler=true
osm-edge installed successfully in namespace [osm-system] with mesh name [osm]
```

