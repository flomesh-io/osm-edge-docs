---
title: 「osm-edge コントロール プレーンとコンポーネントをアンインストールする」
description: "アンインストール"
type: docs
weight: 4
---

# osm-edge コントロール プレーンとコンポーネントのアンインストール

このガイドでは、Kubernetes クラスターから osm-edge をアンインストールする方法について説明します。 このガイドでは、単一の osm-edge コントロール プレーン (メッシュ) が実行されていることを前提としています。 クラスター内に複数のメッシュがある場合は、ガイドの最後にあるクラスター全体のリソースをアンインストールする前に、クラスター内の各コントロール プレーンに対して説明されているプロセスを繰り返します。 コントロール プレーンとデータ プレーンの両方を考慮して、このガイドは最小限のダウンタイムで osm-edge のすべての残りの部分をアンインストールすることを目的としています。

## 前提条件

- osm-edge がインストールされた Kubernetes クラスター
- `kubectl` CLI
- [`osm` CLI](/docs/install/#set-up-the-osm-cli) または Helm 3 CLI

## アプリケーション ポッドと Pipy シークレットから Pipy サイドカーを削除する

osm-edge をアンインストールする最初のステップは、アプリケーション ポッドから Pipy サイドカー コンテナーを削除することです。 サイドカー コンテナーは、トラフィック ポリシーを適用します。 それらがない場合、[Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) が適用されていない限り、トラフィックはデフォルトの Kubernetes ネットワーキングに従って Pod との間で流れます。

osm-edge Pipy サイドカーと関連するシークレットは、次の手順で削除されます。

1. [Disable automatic sidecar injection](#disable-automatic-sidecar-injection)
1. [Restart pods](#restart-pods)

### 自動サイドカー インジェクションを無効にする

osm-edge 自動サイドカー インジェクションは、「osm」CLI を介してメッシュに名前空間を追加することによって最も一般的に有効になります。 `osm` CLI を使用して、
名前空間ではサイドカー インジェクションが有効になっています。 複数のコントロール プレーンがインストールされている場合は、必ず「--mesh-name」フラグを指定してください。

メッシュ内の名前空間を表示:

```console
$ osm namespace list --mesh-name=<mesh-name>
NAMESPACE          MESH           SIDECAR-INJECTION
<namespace1>       <mesh-name>    enabled
<namespace2>       <mesh-name>    enabled
```

メッシュから各名前空間を削除します。

```console
$ osm namespace remove <namespace> --mesh-name=<mesh-name>
Namespace [<namespace>] successfully removed from mesh [<mesh-name>]
```

これにより、名前空間から「openservicemesh.io/sidecar-injection: enabled」アノテーションと「openservicemesh.io/monitored-by: <メッシュ名>」ラベルが削除されます。

または、名前空間ごとではなくポッドのアノテーションを介してサイドカー インジェクションが有効になっている場合は、ポッドまたはデプロイ仕様を変更して、サイドカー インジェクションのアノテーションを削除してください。

### Pod を再起動する

サイドカーで実行されているすべてのポッドを再起動します。

```console
# If pods are running as part of a Kubernetes deployment
# Can use this strategy for daemonset as well
$ kubectl rollout restart deployment <deployment-name> -n <namespace>

# If pod is running standalone (not part of a deployment or replica set)
$ kubectl delete pod <pod-name> -n namespace
$ k apply -f <pod-spec> # if pod is not restarted as part of replicaset
```

これで、以前はメッシュの一部であったアプリケーションの一部として実行されている osm-edge Pipy サイドカー コンテナーはなくなります。 トラフィックはありません
上記で使用された `mesh-name` を持つ osm-edge コントロール プレーンによって管理されなくなりました。 このプロセス中に、アプリケーションでダウンタイムが発生する場合があります
すべての Pod が再起動しているためです。

## osm-edge コントロール プレーンをアンインストールし、ユーザー提供のリソースを削除する

osm-edge コントロール プレーンと関連コンポーネントは、次の手順でアンインストールされます。

1. [Uninstall the osm-edge control plane](#uninstall-the-osm-control-plane)
1. [Remove User Provided Resources](#remove-user-provided-resources)
1. [Delete osm-edge Namespace](#delete-osm-namespace)
1. [Removal of osm-edge Cluster Wide Resources](#removal-of-osm-cluster-wide-resources)

### osm-edge コントロール プレーンをアンインストールする

「osm」CLI を使用して、Kubernetes クラスターから osm-edge コントロール プレーンをアンインストールします。 次の手順で削除します。

1. osm-edge コントローラー リソース (デプロイ、サービス、メッシュ構成、および RBAC)
1. osm-edge によってインストールされる Prometheus、Grafana、Jaeger、および Fluent Bit リソース
1. Webhook の変更と Webhook の検証
1. osm-edge によってインストール/要求される CRD に osm-edge によってパッチされた変換 Webhook フィールド: [osm-edge の CRD](https://github.com/flomesh-io/osm-edge/tree/{{ < param osm_branch >}}/cmd/osm-bootstrap/crds) にはパッチが適用されません。 クラスター全体のリソースを削除するには、詳細について [osm-edge クラスター全体のリソースの削除](#removal-of-osm-cluster-wide-resources) を参照してください。

`osm uninstall mesh` を実行します:

```console
# Uninstall osm control plane components
$ osm uninstall mesh --mesh-name=<mesh-name>
Uninstall osm-edge [mesh name: <mesh-name>] ? [y/n]: y
osm-edge [mesh name: <mesh-name>] uninstalled
```

その他のオプションについては、「osm uninstall mesh --help」を実行してください。

または、Helm を使用してコントロール プレーンをインストールした場合は、次の「helm uninstall」コマンドを実行します。

```console
$ helm uninstall <mesh name> --namespace <osm namespace>
```

その他のオプションについては、`helm uninstall --help` を実行してください。

### ユーザー提供のリソースを削除する

インストール時に osm-edge 用にリソースが提供または作成された場合は、この時点で削除できます。

たとえば、[Hashicorp Vault](/docs/guides/certificates/#installing-hashi-vault) が osm-edge の証明書を管理するためだけにデプロイされた場合、関連するすべてのリソースを削除できます。

### osm-edge 名前空間を削除

メッシュをインストールするとき、「osm」CLI は、コントロール プレーンがインストールされる名前空間がまだ存在しない場合に作成します。 ただし、同じメッシュをアンインストールする場合、存在する名前空間は `osm` CLI によって自動的に削除されません。 この動作は、次の理由で発生します。
ユーザーが名前空間に作成したリソースが、自動的に削除されたくない場合があります。

名前空間が osm-edge にのみ使用され、保持する必要があるものが何もない場合、アンインストール時または後で次のコマンドを使用して名前空間を削除できます。

```console
$ osm uninstall mesh --delete-namespace
```

> 警告: 名前空間のリソースが不要になった場合にのみ、名前空間を削除してください。 たとえば、osm が「kube-system」にインストールされている場合、名前空間を削除すると、重要なクラスター リソースが削除され、意図しない結果が生じる可能性があります。


### osm-edge クラスター全体のリソースの削除

インストール時に、osm-edge は　[ここ](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/cmd/osm-bootstrap/crds) に記載されているすべての CRD を保証します。 インストール時にクラスターに存在します。 インストール中に、それらがまだインストールされていない場合、「osm-bootstrap」ポッドは、残りのコントロール プレーン コンポーネントが実行される前にそれらをインストールします。 これは、Helm チャートを使用して osm-edge をインストールする場合も同じ動作です。 

管理されていない環境と管理されている環境の両方でのメッシュのアンインストール:
1. コントロール プレーン ポッドを含む、osm-edge コントロール プレーン コンポーネントを削除します。
2. すべての CRD (複数の CR バージョンをサポートするために osm-edge が追加) から変換 Webhook フィールドを削除/パッチ解除します。

osm-edge のアンインストール後にクラスターに意図しない結果が生じるのを防ぐために、特定の osm-edge リソースを残します。残されるリソースは、osm-edge が管理されたクラスター環境からアンインストールされたか、管理されていないクラスター環境からアンインストールされたかによって異なります。

osm-edge をアンインストールするとき、「osm uninstall mesh」コマンドと Helm アンインストールの両方が、主に次の 2 つの理由から、どのクラスター環境 (管理対象および非管理対象) でも osm-edge または SMI CRD を削除しません。
1. CRD はクラスター全体のリソースであり、同じクラスターで実行されている他のサービス メッシュまたはリソースによって使用される場合があります。
2. CRD を削除すると、その CRD に対応するすべてのカスタム リソースも削除されます。

osm-edge がインストールするクラスター全体のリソース (つまり、meshconfig、secrets、osm-edge CRD、SMI CRD、および webhook 構成) を削除するには、osm-edge のインストール解除中またはインストール後に次のコマンドを実行できます。

```bash
osm uninstall mesh --delete-cluster-wide-resources
```

> 警告: CRD を削除すると、その CRD に対応するすべてのカスタム リソースも削除されます。

osm-edge アンインストールのトラブルシューティングについては、[uninstall troubleshooting section](/docs/guides/troubleshooting/uninstall/) を参照してください。