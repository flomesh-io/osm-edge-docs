---
title: 「外部認証」
description: 「MeshConfig を使用して外部認証を構成します。」
type: docs
weight: 1
---
# 外部認可

## 概要
外部承認により、必ずしもプロキシのコンテキスト内で実行されているとは限らないリモート エンドポイントへの着信 HTTP 要求の承認をオフロードできます。
外部承認をサポートするプロキシは、すべての要求に対して、リモート エンドポイントにアクセス許可チェックを発行します。 システムが持つ可能性のある RPS の数が多い可能性があるため、外部認証には、パフォーマンスを犠牲にして多かれ少なかれ制御できるようにするための調整可能な設定がいくつかあります (HTTP ペイロードまたはヘッダーのみを送信する、タイムアウト、認証が応答しない場合のデフォルトの動作、 等。）。

osm-edge は、osm-edge の MeshConfig を介して envoy の [外部認証拡張機能](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter) を構成できます。

## 制限事項
現在、承認フィルタリングは、ユーザーが定義したプロキシのサブセットまたはグループにインストールするように動的に構成することはできません。代わりに、有効にすると、メッシュ内のすべてのサービスにグローバルに適用されます。

同様に、フィルタリング方向は、メッシュ内の「インバウンド」および「イングレス」接続に静的に適用され、有効にするとメッシュ内のサービスまたはアプリケーションに対して行われるすべての HTTP リクエストに影響を与えます。


## OPA プラグインを使用した osm-edge 外部認証のウォークスルー
次のセクションでは、「opa-envoy-plugin」と組み合わせて外部認証を構成する方法について説明します。

[OPA envoy plugin's](https://github.com/open-policy-agent/opa-envoy-plugin)のドキュメントを最初に読んで、構成オプションと展開モデルをさらに理解することを強くお勧めします。

次の例では、単一のリモート (ネットワーク経由) エンドポイントを使用して、すべてのトラフィックを検証します。 この構成は、実動展開には推奨されません。

- まず、osm-edge のデモをデプロイすることから始めます。 このサンプル展開を使用して、外部認証機能をテストします。 [osm-edge's Automated Demo](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/demo#how-to-run-the-osm-automated-demo)を参照し、指示に従ってください。

```bash
# Assuming osm-edge repo is available
cd <PATH_TO_OSM_REPO>
demo/run-osm-demo.sh  # wait for all services to come up
```

- osm-edge のデモが稼働したら、「opa-envoy-plugin」のデプロイに進みます。 osm-edge は [精選されたスタンドアロンの opa-envoy-plugin デプロイメント チャート](https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/opa/deploy-opa- envoy.yaml) は、サービスを介してネットワーク経由で「opa-envoy-plugin」の gRPC ポート (デフォルトは「9191」) を公開します。 これは、外部認証を有効にするときに osm-edge がプロキシを構成するエンドポイントです。 次のスニペットは `opa` 名前空間を作成し、最小限の全拒否構成で `opa-envoy-plugin` をデプロイします。

```bash
kubectl create namespace opa
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/opa/deploy-opa-envoy.yaml
```

- osm-edge のデモが稼働したら、osm-edge の MeshConfig の編集に進み、メッシュに外部認証を追加します。 そのためには、次のように「inboundExternalAuthorization」を構成して、リモートの外部認証エンドポイントを指すようにします。

```bash
kubectl edit meshconfig osm-mesh-config -n osm-system
## <scroll to the following section>
...
inboundExternalAuthorization:
      enable: true
      address: opa.opa.svc.cluster.local
      port: 9191
      failureModeAllow: false
      statPrefix: inboundExtAuthz
      timeout: 1s
...
```

- このステップの後、osm-edge はすべてのプロキシーを構成して、承認の決定を外部の承認サービスに依存させる必要があります。 デフォルトでは、「opa-envoy-plugin」で提供される構成は、メッシュ内のすべてのサービスへのすべてのリクエストを拒否します。 これは、ネットワーク上の任意のサービスのログで確認できます。「403 Forbidden」が予想されます。
```bash
kubectl logs <bookbuyer_pod> -n bookbuyer bookbuyer
```
```console
...
--- bookbuyer:[ 8 ] -----------------------------------------

Fetching http://bookstore.bookstore:14001/books-bought
Request Headers: map[Client-App:[bookbuyer] User-Agent:[Go-http-client/1.1]]
Identity: n/a
Booksbought: n/a
Server: envoy
Date: Tue, 04 May 2021 01:20:39 GMT
Status: 403 Forbidden
ERROR: response code for "http://bookstore.bookstore:14001/books-bought" is 403;  expected 200
...
```

- 外部認証が機能していることをさらに確認するには、「opa-envoy-plugin」インスタンスのログを確認します。 インスタンスのログには、受信したすべてのリクエストの評価を記録する json blob が含まれている必要があります。キーと値の「result」は、「opa-envoy-plugin」インスタンスに供給された構成によって行われた承認の決定に存在する必要があります。
```bash
kubectl logs <opa_pod_name> -n opa
...
{"decision_id":"1df154b5-658a-47bf-ac18-be52998605da"
...
"result":false,   // resulting decision
...
"time":"2021-05-04T01:21:18Z","timestamp":"2021-05-04T01:21:18.1971808Z","type":"openpolicyagent.org/decision_logs"
}
```

- 前のステップが確認されたら、デフォルトですべてのトラフィックを承認するように OPA ポリシー構成を変更します。

```bash
kubectl edit configmap opa-policy -n opa
...
...
default allow = false
```
Change the previous line value to `true`:
```console
default allow = true
```

- 最後に opa-envoy-plugin を再起動します。 これは、アプリケーションが取得するのではなく、この展開の構成が構成としてプッシュされるため、必要な手順です。
```bash
kubectl rollout restart deployment opa -n opa
```

- 「bookbuyer」サービスへのリクエストが承認されていることを確認します。
```console
--- bookbuyer:[ 2663 ] -----------------------------------------

Fetching http://bookstore.bookstore:14001/books-bought
Request Headers: map[Client-App:[bookbuyer] User-Agent:[Go-http-client/1.1]]
Identity: bookstore-v1
Booksbought: 1087
Server: envoy
Date: Tue, 04 May 2021 02:00:46 GMT
Status: 200 OK
MAESTRO! THIS TEST SUCCEEDED!

Fetching http://bookstore.bookstore:14001/buy-a-book/new
Request Headers: map[]
Identity: bookstore-v1
Booksbought: 1088
Server: envoy
Date: Tue, 04 May 2021 02:00:47 GMT
Status: 200 OK
ESC[90m2:00AMESC[0m ESC[32mINFESC[0m BooksCountV1=21490056 ESC[36mcomponent=ESC[0mdemo ESC[36mfile=ESC[0mbooks.go:167
MAESTRO! THIS TEST SUCCEEDED!
```

OPA プラグインの出力を再検証して、各リクエストに対してポリシーの決定が行われていることを確認できます。
```
{"decision_id":"3f29d449-7f71-4721-b93c-ad7d375e0f80",
...
"result":true,
...
"time":"2021-05-04T02:01:35Z","timestamp":"2021-05-04T02:01:35.816768454Z","type":"openpolicyagent.org/decision_logs"}
```


この例のルールは、OPA プラグインを使用してネットワーク トラフィックに承認ポリシーを適用する方法を示しており、本番環境へのデプロイを目的としたものではありません。 たとえば、この方法ですべての HTTP 呼び出しに承認を適用すると、かなりのオーバーヘッドが追加されますが、OPA インジェクターを使用して削減できます。 OPA の使用に関する詳細については、[Open Policy Agent official documentation](https://www.openpolicyagent.org/docs/latest/) を参照してください。