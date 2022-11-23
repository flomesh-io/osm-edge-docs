---
title: 「サーキットブレイキング」
description: 「サーキット ブレーカーを使用して接続とリクエストを制限する」
type: docs
weight: 11
---

# サーキットブレーカー

 サーキット ブレーカーは、分散システムの重要なコンポーネントであり、重要な回復パターンです。 サーキット ブレーカーを使用すると、アプリケーションがすぐに失敗し、できるだけ早くダウンストリームにバック プレッシャーを適用できるようになります。これにより、システム全体の障害の影響を制限する手段が提供されます。 このガイドでは、osm-edge でサーキット ブレーカーを構成する方法について説明します。
## 回路遮断の構成
osm-edge は、その [UpstreamTrafficSetting API][1] を利用して、アップストリーム サービスに向けられたトラフィックのサーキット ブレーカー属性を構成します。 「アップストリーム サービス」という用語は、クライアントからの接続と要求を受け取り、応答を返すサービスを指すために使用します。 この仕様では、接続および要求レベルでアップストリーム サービスのサーキット ブレーカー属性を構成できます。

各「UpstreamTrafficSetting」構成は、「spec.host」フィールドで定義されたアップストリーム ホストを対象としています。 名前空間「my-namespace」の Kubernetes サービス「my-svc」の場合、名前空間「my-namespace」に「UpstreamTrafficSetting」リソースを作成する必要があり、「spec.host」は「my」形式の FQDN である必要があります。 -svc.my-namespace.svc.cluster.local`．[Egress policy](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.EgressSpec)　で一致として指定された場合、「spec.host」は Egress ポリシーで指定されたホストに対応している必要があります。 「UpstreamTrafficSetting」構成は、「Egress」リソースと同じ名前空間に属している必要があります。

サーキット ブレーカーは、TCP と HTTP の両方のレベルで適用でき、「UpstreamTrafficSetting」リソースの「connectionSettings」属性を使用して構成できます。 TCP トラフィック設定は TCP トラフィックと HTTP トラフィックの両方に適用されますが、HTTP 設定は HTTP トラフィックにのみ適用されます。

次の回路遮断構成がサポートされています。

- `Maximum connections`: `UpstreamTrafficSetting` 構成の `spec.host` フィールドで指定されたアップストリーム ホストに属するすべてのバックエンドに対してクライアントが確立できる接続の最大数。この設定は「tcp.maxConnections」フィールドを使用して構成でき、TCP と HTTP トラフィックの両方に適用できます。指定しない場合、デフォルトは `4294967295` (2^32 - 1) です。

- `Maximum pending requests`: キューに入れることを許可されているアップストリーム ホストへの保留中の HTTP 要求の最大数。要求をすぐにディスパッチするのに十分なアップストリーム接続がない場合は常に、保留中の要求のリストに要求が追加されます。 HTTP/2 接続の場合、「http.maxRequestsPerConnection」が構成されていない場合、すべての要求が同じ接続を介して多重化されるため、このサーキット ブレーカーは、接続がまだ確立されていない場合にのみヒットします。この設定は、「http.maxPendingRequests」フィールドを使用して構成でき、HTTP トラフィックにのみ適用されます。指定しない場合、デフォルトは `4294967295` (2^32 - 1) です。

- `Maximum requests`: クライアントがアップストリーム ホストに対して行うことができる並列リクエストの最大数。 この設定は「http.maxRequests」フィールドを使用して構成でき、HTTP トラフィックにのみ適用されます。 指定しない場合、デフォルトは `4294967295` (2^32 - 1) です。

- `接続ごとの最大リクエスト数`: 接続ごとに許可されるリクエストの最大数。 この設定は「http.maxRequestsPerConnection」フィールドを使用して構成でき、HTTP トラフィックにのみ適用されます。 指定がない場合は無制限です。

- `Maximum active retries`: クライアントがアップストリーム ホストに対して実行できるアクティブな再試行の最大数。 この設定は、「http.maxRetries」フィールドを使用して構成でき、HTTP トラフィックにのみ適用されます。 指定しない場合、デフォルトは `4294967295` (2^32 - 1) です。


サーキット ブレーカーの構成の詳細については、次のデモ ガイドを参照してください。
- [Circuit breaking for destinations within the mesh](/docs/demos/circuit_breaking_mesh_internal)
- [Circuit breaking for destinations external to the mesh](/docs/demos/circuit_breaking_mesh_external)

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec