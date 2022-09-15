---
title: 「サイドカーインジェクション」
description: "このセクションでは、osm-edge でのサイドカー インジェクションのワークフローについて説明します。"
type: docs
weight: 3
---

# サイドカー インジェクション
サービス メッシュに参加するサービスは、サービスをサポートするポッドにインストールされたサイドカー プロキシを介して通信します。 次のセクションでは、osm-edge でのサイドカー インジェクションのワークフローについて説明します。

## 自動サイドカー インジェクション
自動サイドカー インジェクションは現在、サービス メッシュにサイドカーをインジェクトする唯一の方法です。 サイドカーは、osm-edge によって提供される変更 Webhook アドミッション コントローラーを使用して、該当する Kubernetes ポッドに自動的に挿入できます。

自動サイドカー インジェクションは、名前空間をメッシュに登録する一環として、または後で Kubernetes API を使用して、名前空間ごとに構成できます。 自動サイドカー インジェクションは、ネームスペースまたはポッド リソースにサイドカー インジェクション アノテーションを付けることで、ネームスペースごとまたはポッドごとに有効にすることができます。 個々のポッドと名前空間は、自動サイドカー インジェクションを有効または無効にするように明示的に構成できるため、ユーザーはポッドと名前空間でのサイドカー インジェクションを柔軟に制御できます。

### 自動サイドカー インジェクションの有効化

前提条件:
- Pod が属する名前空間は、「osm namespace add」コマンドを使用してメッシュに追加された監視対象の名前空間でなければなりません。
- Pod が属する名前空間は、`osm namespace ignore` コマンドを使用して無視されるように設定してはなりません。
- Pod が属する名前空間には、osm-edge コントロール プレーンの名前空間に対応するキー「name」と値を持つラベルがあってはなりません。 たとえば、「名前: osm-system」というラベルが付いた名前空間では、「osm-system」がコントロール プレーンの名前空間であり、この名前空間のポッドに対してサイドカー インジェクションを有効にすることはできません。
- ポッドの仕様に「hostNetwork: true」が含まれていてはなりません。 ホスト ネットワークでルーティング エラーが発生する可能性があるため、「hostNetwork: true」を持つ Pod にはサイドカーが挿入されません。

自動サイドカー インジェクションは、次の方法で有効にできます。

- `osm` cli を使用して名前空間をメッシュに登録している間: `osm namespace add <namespace>`:
   このコマンドを使用すると、自動サイドカー インジェクションがデフォルトで有効になります。

- `kubectl` を使用して個々の名前空間とポッドに注釈を付け、サイドカー インジェクションを有効にします。

  ```console
  # Enable sidecar injection on a namespace
  $ kubectl annotate namespace <namespace> openservicemesh.io/sidecar-injection=enabled
  ```

  ```console
  # Enable sidecar injection on a pod
  $ kubectl annotate pod <pod> openservicemesh.io/sidecar-injection=enabled
  ```

- 名前空間またはポッドの Kubernetes リソース仕様で、サイドカー インジェクション アノテーションを「有効」に設定します。
  ```yaml
  metadata:
    name: test
    annotations:
      'openservicemesh.io/sidecar-injection': 'enabled'
  ```

  次の条件が満たされている場合にのみ、Pod にサイドカーが挿入されます。
   1. Pod が属する名前空間は、監視対象の名前空間です。
   2. ポッドがサイドカー インジェクションに対して明示的に有効になっている、またはポッドが属する名前空間でサイドカー インジェクションが有効になっていて、ポッドがサイドカー インジェクションに対して明示的に無効になっていない。

### 名前空間での自動サイドカー インジェクションの明示的な無効化

次の方法で、自動サイドカー インジェクションに対して名前空間を無効にすることができます。

- `osm` cli を使用して名前空間をメッシュに登録している間: `osm namespace add <namespace> --disable-sidecar-injection`:
   名前空間でサイドカー インジェクションが以前に有効化されていた場合は、このコマンドの実行後に無効化されます。

- `kubectl` を使用して個々の名前空間に注釈を付け、サイドカー インジェクションを無効にします。

  ```console
  # Disable sidecar injection on a namespace
  $ kubectl annotate namespace <namespace> openservicemesh.io/sidecar-injection=disabled
  ```

### Pod での自動サイドカー インジェクションの明示的な無効化

サイドカー インジェクションに対して個々のポッドを明示的に無効にすることができます。 これは、名前空間でサイドカー インジェクションが有効になっているが、特定のポッドにサイドカーをインジェクションするべきではない場合に役立ちます。

- `kubectl` を使用して個々の Pod にアノテーションを付け、サイドカー インジェクションを無効にします。
  ```console
  # Disable sidecar injection on a pod
  $ kubectl annotate pod <pod> openservicemesh.io/sidecar-injection=disabled
  ```

- ポッドの Kubernetes リソース仕様で、サイドカー インジェクション アノテーションを「無効」に設定します。
  ```yaml
  metadata:
    name: test
    annotations:
      'openservicemesh.io/sidecar-injection': 'disabled'
  ```

「osm namespace remove」コマンドを使用してメッシュから名前空間を削除すると、名前空間の自動サイドカー インジェクションが暗黙的に無効になります。
