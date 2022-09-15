---
title: 「osm-edge コントロール プレーンをインストールする」
description: 「このセクションでは、Kubernetes クラスターに osm-edge をインストール/アンインストールする方法について説明します」
type: docs
weight: 2
---

## 前提条件

- Kubernetes {{< param min_k8s_version >}} 以上を実行している Kubernetes クラスター
- [osm-edge CLI](/docs/guides/cli) または [helm 3 CLI](https://helm.sh/docs/intro/install/) または OpenShift `oc` CLI。

### Kubernetes のサポート

osm-edge は、osm-edge リリース時にサポートされている Kubernetes バージョンで実行できます。 現在のサポート マトリックスは次のとおりです。

| osm-edge          | Kubernetes  |
| ----------------- | ----------- |
| 1.1               | 1.19 - 1.24 |

### osm-edge CLI の使用

「osm」CLI を使用して、osm-edge コントロール プレーンを Kubernetes クラスターにインストールします。

#### osm-edge CLI とチャートの互換性

osm-edge CLI の各バージョンは、一致するバージョンの osm-edge Helm チャートでのみ動作するように設計されています。 バージョンの偏りが存在する場合でも、多くの操作は引き続き機能する可能性がありますが、これらのシナリオはテストされておらず、異なる CLI およびチャート バージョンを使用するときに発生する問題は、報告されても修正されない場合があります。

#### CLI の実行

「osm install」を実行して、osm-edge コントロール プレーンをインストールします。

```console
$ osm install
osm-edge installed successfully in namespace [osm-system] with mesh name [osm]
```

その他のオプションについては、「osm install --help」を実行してください。

_注: CLI を使用して osm-edge をインストールすると、クラスター内にメッシュを 1 つだけ展開することが強制されます。 osm-edge は、CRD を osm-edge の特定のインスタンスに結び付ける複数の API バージョンをサポートするために、変換 Webhook フィールドをすべての CRD に追加することにより、CRD をインストールおよび管理します。 したがって、osm-edge の正しい操作のために、クラスタごとに 1 つの osm-edge メッシュのみを持つことが **強く推奨されます**。_

### Helm CLI の使用

[osm-edge チャート](https://github.com/flomesh-io/osm-edge/tree/{{< param osm_branch >}}/charts/osm) は [Helm CLI]( https://helm.sh/docs/intro/install/)。

#### 値ファイルの編集

値ファイルをオーバーライドすることで、osm-edge インストールを構成できます。

1. [values file](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/values.yaml) のコピーを作成します (必ず使用してください インストールするチャートのバージョン)。
1. カスタマイズしたい値を変更します。 その他の値はすべて省略できます。

    - MeshConfig 設定に対応する値を確認するには、[osm-edge MeshConfig documentation](/docs/guides/mesh_config) を参照してください。

    - たとえば、MeshConfig の「logLevel」フィールドを「info」に設定するには、以下を「override.yaml」として保存します。
     ```console
     osm:
       sidecarLogLevel: info
     ```

#### ヘルムのインストール

次に、次の「helm install」コマンドを実行します。 チャートのバージョンは、[こちら](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/Chart) にインストールしたい Helm チャートにあります。 .yaml#L17)。

```console
$ helm install <mesh name> osm --repo https://flomesh-io.github.io/osm-edge --version <chart version> --namespace <osm namespace> --values override.yaml
```

デフォルト設定を使用する場合は、`--values` フラグを省略します。

その他のオプションについては、`helm install --help` を実行してください。

### OpenShift

OpenShift に osm-edge をインストールするには:

1. iptables を適切にプログラミングできるように、特権付きの init コンテナーを有効にします。 NET_ADMIN 機能は、OpenShift では十分ではありません。
   ```bash
   osm install --set="osm.enablePrivilegedInitContainer=true"
   ```
   - 特権 init コンテナーを有効にせずに osm-edge を既にインストールしている場合は、[osm-edge MeshConfig](/docs/guides/mesh_config) で「enablePrivilegedInitContainer」を「true」に設定し、メッシュ内のすべてのポッドを再起動します。
1. `privileged` [security context constraint](https://docs.openshift.com/container-platform/4.7/authentication/managing-security-context-constraints.html) をメッシュ内の各サービス アカウントに追加します。
    - [oc CLI](https://docs.openshift.com/container-platform/4.7/cli_reference/openshift_cli/getting-started-cli.html) をインストールします。
    - セキュリティ コンテキストの制約をサービス アカウントに追加する
     ```bash
      oc adm policy add-scc-to-user privileged -z <service account name> -n <service account namespace>
     ```

### ポッド セキュリティ ポリシー

**非推奨: PSP サポートは v0.10.0 以降、osm-edge で非推奨になっています。**

**PSP サポートは osm-edge 1.0.0 で削除されます**

PSP が有効なクラスターで osm-edge を実行している場合は、`--set osm.pspEnabled=true` を `osm install` または `helm install` CLI コマンドに渡します。

### osm-edge で Reconciler を有効にする

osm-edge でリコンサイラーを有効にしたい場合は、`--set osm.enableReconciler=true` を `osm install` または `helm install` CLI コマンドに渡します。 Reconciler の詳細については、[Reconciler Guide](/docs/guides/reconciler) を参照してください。

## osm-edge コンポーネントの検査

デフォルトでいくつかのコンポーネントがインストールされます。 次の「kubectl」コマンドを使用して検査します。

```console
# Replace osm-system with the namespace where osm-edge is installed
$ kubectl get pods,svc,secrets,meshconfigs,serviceaccount --namespace osm-system
```

いくつかのクラスター全体 (ネームスペース化されていないコンポーネント) もインストールされます。 次の「kubectl」コマンドを使用して検査します。

```console
kubectl get clusterrolebinding,clusterrole,mutatingwebhookconfiguration,validatingwebhookconfigurations -l app.kubernetes.io/name=openservicemesh.io
```

内部では、`osm` は [Helm](https://helm.sh) ライブラリを使用して、コントロール プレーンの名前空間に Helm の `release` オブジェクトを作成しています。 Helm の「リリース」名はメッシュ名です。 `helm` CLI を使用して、インストールされた Kubernetes マニフェストをより詳細に検査することもできます。 Helm のインストール手順については、https://helm.sh にアクセスしてください。

```console
# Replace osm-system with the namespace where osm-edge is installed
$ helm get manifest osm --namespace osm-system
```

## 次のステップ

osm-edge コントロール プレーンが起動して実行されるようになったので、メッシュに [add services](/docs/guides/app_onboarding/) します。
