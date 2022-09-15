---
title: 「プロメテウスのトラブルシューティング」
description: 「osm-edge の Prometheus 統合に関する一般的な問題を修正する方法」
aliases: "/docs/troubleshooting/observability/prometheus"
type: docs
weight: 2
---

## プロメテウスに到達できません

osm-edge でインストールされた Prometheus インスタンスに到達できない場合は、次の手順を実行して、問題を特定して解決します。

1. Prometheus Pod が存在することを確認します。

     「osm install --set=osm.deployPrometheus=true」でインストールすると、「osm-prometheus-5794755b9f-rnvlr」のような名前の Prometheus Pod が、「osm」という名前の他の osm-edge コントロール プレーン コンポーネントの名前空間に存在する必要があります。 -system` デフォルトで。

     そのような Pod が見つからない場合は、osm-edge Helm チャートが、`helm` で `osm.deployPrometheus` パラメータを `true` に設定してインストールされたことを確認します。

    ```console
    $ helm get values -a <mesh name> -n <osm-edge namespace>
    ```

    パラメータが「true」以外に設定されている場合は、「osm install」で「--set=osm.deployPrometheus=true」フラグを使用して osm-edge を再インストールします。

1. Prometheus Pod が正常であることを確認します。

     上記の Prometheus Pod は、「kubectl get」の出力に示されているように、実行中の状態であり、すべてのコンテナーの準備が整っている必要があります。

    ```console
    $ # Assuming osm-edge is installed in the osm-system namespace:
    $ kubectl get pods -n osm-system -l app=osm-prometheus
    NAME                              READY   STATUS    RESTARTS   AGE
    osm-prometheus-5794755b9f-67p6r   1/1     Running   0          27m
    ```

    Pod が Running またはそのコンテナーの準備ができていると表示されない場合は、「kubectl describe」を使用して他の潜在的な問題を探します。

    ```console
    $ # Assuming osm-edge is installed in the osm-system namespace:
    $ kubectl describe pods -n osm-system -l app=osm-prometheus
    ```

    Prometheus Pod が正常であることが確認されると、Prometheus に到達できるようになります。

## Prometheus に指標が表示されない

Prometheus が Pod のメトリクスをスクレイピングしていないことが判明した場合は、次の手順を実行して問題を特定して解決します。

1. アプリケーション Pod が期待どおりに動作していることを確認します。

     メッシュで実行されているワークロードが適切に機能していない場合、それらの Pod からスクレイピングされたメトリックが正しく表示されない可能性があります。 たとえば、サービス B からサービス A へのトラフィックを示すメトリクスがない場合は、サービスが正常に通信していることを確認します。

     この種の問題をさらにトラブルシューティングするには、トラフィックのトラブルシューティング ガイドを参照してください。

1. メトリクスが欠落している Pod に Pipy サイドカーが挿入されていることを確認します。

     Pipy サイドカー コンテナーを備えた Pod のみが、Prometheus によってメトリックがスクレイピングされることが期待されます。 各 Pod が、名前に「flomesh/pipy」を含むイメージからコンテナーを実行していることを確認します。

    ```console
    $ kubectl get po -n <pod namespace> <pod name> -o jsonpath='{.spec.containers[*].image}'
    mynamespace/myapp:v1.0.0 flomesh/pipy:0.50.0
    ```
1. Prometheus によってスクレイピングされているプロキシのエンドポイントが期待どおりに機能していることを確認します。

     各 Pipy プロキシは、そのプロキシによって生成され、Prometheus によってスクレイピングされるメトリックを示す HTTP エンドポイントを公開します。 エンドポイントに直接リクエストを送信して、予想されるメトリックが表示されるかどうかを確認します。

     メトリクスが欠落している Pod ごとに、「kubectl」を使用して Pipy プロキシ管理インターフェイス ポートを転送し、メトリクスを確認します。

    ```console
    $ kubectl port-forward -n <pod namespace> <pod name> 15000
    ```

    ブラウザーで http://localhost:15000/stats/prometheus に移動して、その Pod によって生成されたメトリックを確認します。 Prometheus がこれらのメトリクスを考慮していないように思われる場合は、次のステップに進み、Prometheus が適切に構成されていることを確認してください。

1. 目的の名前空間がメトリック コレクションに登録されていることを確認します。

     メトリックスクレイピングが必要な Pod を含む名前空間ごとに、「osm mesh list」を使用して目的の osm-edge インスタンスによって名前空間が監視されていることを確認します。

     次に、名前空間に「openservicemesh.io/metrics: enabled」という注釈が付けられていることを確認します。

    ```console
    $ # Assuming osm-edge is installed in the osm-system namespace:
    $ kubectl get namespace <namespace> -o jsonpath='{.metadata.annotations.openservicemesh\.io/metrics}'
    enabled
    ```

    そのようなアノテーションが名前空間に存在しないか、値が異なる場合は、`osm` で修正します。

    ```console
    $ osm metrics enable --namespace <namespace>
    Metrics successfully enabled in namespace [<namespace>]
    ```

2. [カスタム指標](/docs/guides/observability/metrics/#custom-metrics) がスクレイピングされていない場合は、それらが有効になっていることを確認してください。

     カスタム メトリックは現在、デフォルトで無効になっており、`osm.featureFlags.enableWASMStats` パラメータが `true` に設定されている場合に有効になります。 現在の osm-edge インスタンスに、`<osm-namespace>` 名前空間の `<osm-mesh-name>` という名前のメッシュ用にこのパラメーターが設定されていることを確認します。

    ```console
    $ helm get values -a <osm-mesh-name> -n <osm-namespace>
    ```

   > 注: `<osm-mesh-name>` を osm メッシュの名前に置き換え、`<osm-namespace>` を osm がインストールされた名前空間に置き換えます。

    `osm.featureFlags.enableWASMStats` が別の値に設定されている場合は、osm-edge を再インストールし、`--set osm.featureFlags.enableWASMStats` を `osm install` に渡します。
