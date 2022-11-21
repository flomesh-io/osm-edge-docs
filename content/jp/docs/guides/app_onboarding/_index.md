---
title: 「アプリケーション オンボーディング」
description: 「機内サービス」
type: docs
weight:: 5
---

# 機内サービス
次のガイドでは、Kubernetes マイクロサービスを osm-edge インスタンスにオンボードする方法について説明します。

1. アプリケーションをオンボーディングする前に、[アプリケーション要件](docs/guides/app_onboarding/prereqs) ガイドを参照してください。

2. [サービス メッシュ インターフェイス (SMI) ポリシー] の構成とインストール (https://github.com/servicemeshinterface/smi-spec)

    osm-edge は SMI 仕様に準拠しています。 デフォルトでは、SMI ポリシーで明示的に許可されていない限り、osm-edge は Kubernetes サービス間のすべてのトラフィック通信を拒否します。 この動作は、`osm install` コマンドの `--set=osm.enablePermissiveTrafficPolicy=true` フラグでオーバーライドできます。これにより、トラフィックとサービスが mTLS 暗号化などの機能を引き続き利用できるようにしながら、SMI ポリシーを強制しないようにすることができます。 トラフィック、メトリック、およびトレース。

    SMI ポリシーの例については、次の例を参照してください。
    - [demo/deploy-traffic-specs.sh](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/deploy-traffic-specs.sh)
    - [demo/deploy-traffic-split.sh](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/deploy-traffic-split.sh)
    - [demo/deploy-traffic-target.sh](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/deploy-traffic-target.sh)

3. メッシュ内のアプリケーションが Kubernetes API サーバーと通信する必要がある場合、ユーザーは、IP 範囲の除外を使用するか、以下に概説するようにエグレス ポリシーを作成して、これを明示的に許可する必要があります。

   まず、Kubernetes API サーバー クラスターの IP を取得します。
   ```console
   $ kubectl get svc -n default
   NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
   kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   1d
   ```

   **オプション 1:** 除外するグローバル アウトバウンド IP 範囲のリストに Kubernetes API サーバーのアドレスを追加します。 IP アドレスは、クラスター IP アドレスまたはパブリック IP アドレスである可能性があり、Kubernetes API サーバーへの接続のために適切に除外する必要があります。

    この IP を MeshConfig に追加して、そこへのアウトバウンド トラフィックが osm-edge のサイドカーによるインターセプトから除外されるようにします。
    ```console
    $ kubectl patch meshconfig osm-mesh-config -n <osm-namespace> -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["10.0.0.1/32"]}}}'  --type=merge
    meshconfig.config.openservicemesh.io/osm-mesh-config patched
    ```
    
    この変更を有効にするには、監視対象の名前空間で関連するポッドを再起動します。

    **オプション 2:** エグレス ポリシーを適用して、HTTPS 経由で Kubernetes API サーバーへのアクセスを許可する
   
   > _注: Egress ポリシーを使用する場合、Kubernetes API サービスは、osm-edge が管理する名前空間に存在してはなりません_

    1. 有効になっていない場合は、エグレス ポリシーを有効にします。
    ```console
    kubectl patch meshconfig osm-mesh-config -n <osm-namespace> -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
    ```
   
    2. Egress ポリシーを適用して、アプリケーションの ServiceAccount が上記の Kubernetes API サーバー クラスター IP にアクセスできるようにします。
     例えば: 
    ```console
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
        name: k8s-server-egress
        namespace: test
    spec:
        sources:
        - kind: ServiceAccount
          name: <app pod's service account name>
          namespace: <app pod's service account namespace>
        ipAddresses:
        - 10.0.0.1/32
        ports:
        - number: 443
          protocol: https
    EOF
    ```  

4. osm-edge に Kubernetes 名前空間をオンボードする

    osm-edge によって管理されるアプリケーションを含む名前空間をオンボードするには、「osm namespace add」コマンドを実行します。

    ```console
    $ osm namespace add <namespace> --mesh-name <mesh-name>
    ```

    デフォルトでは、「osm namespace add」コマンドは、名前空間内のポッドの自動サイドカー インジェクションを有効にします。

     名前空間をメッシュに登録する一環として自動サイドカー インジェクションを無効にするには、「osm namespace add <namespace> --disable-sidecar-injection」を使用します。
     名前空間がオンボーディングされると、自動サイドカー インジェクションを構成することで、ポッドをメッシュに登録できます。 詳細については、[サイドカー インジェクション](docs/guides/app_onboarding/sidecar_injection) ドキュメントを参照してください。

5. 新しいアプリケーションをデプロイするか、既存のアプリケーションを再デプロイする

    デフォルトでは、オンボードされた名前空間での新しいデプロイは、自動サイドカー インジェクションに対して有効になっています。 これは、管理された名前空間で新しい Pod が作成されると、osm-edge が自動的にサイドカー プロキシを Pod に挿入することを意味します。
     Pod の再作成時に osm-edge が自動的にサイドカー プロキシを挿入できるように、既存のデプロイメントを再起動する必要があります。 Deployment によって管理される Pod は、「kubectl rollout restart deploy」コマンドを使用して再起動できます。

    プロトコル固有のトラフィックをサービス ポートに正しくルーティングするには、使用するアプリケーション プロトコルを構成します。 詳細については、[アプリケーション プロトコル選択ガイド](docs/guides/app_onboarding/app_protocol_selection)を参照してください。

#### 注: 名前空間の削除
「osm namespace remove」コマンドを使用して、osm-edge メッシュから名前空間を削除できます。
```console
$ osm namespace remove <namespace>
```

> **ご注意ください: **
> **`osm namespace remove`** コマンドは、名前空間内のサイドカー プロキシ構成への更新の適用を停止するように osm-edge に指示するだけです。 **プロキシ サイドカーは削除されません**。 これは、既存のプロキシ構成が引き続き使用されることを意味しますが、osm-edge コントロール プレーンによって更新されることはありません。 すべてのポッドからプロキシを削除する場合は、CLI を使用して osm-edge メッシュからポッドの名前空間を削除し、すべてのポッド ワークロードを再インストールします。
