---
title: 「トラフィック分割」
description: 「SMI Traffic Split API を使用したトラフィック分割」
type: docs
weight: 10
---

# トラフィック分割

[SMI Traffic Split API](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md) を使用して、発信トラフィックを複数のサービスに分割できます バックエンド。 これは、ソフトウェアの複数のバージョンのカナリア リリースを調整するために使用できます。

## サポート対象

osm-edge は [SMI traffic split v1alpha2 version](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md) を実装します.

以下をサポートします。

- SMI および Permissive トラフィック ポリシー モードの両方でのトラフィック分割
- HTTP と TCP のトラフィック分割
- カナリアまたはブルー グリーン デプロイのトラフィック分割

## 使い方

Kubernetes サービス宛てのアウトバウンド トラフィックは、SMI トラフィック分割 API を使用して複数のサービス バックエンドに分割できます。 `default/bookstore` サービスに対応する `bookstore.default.svc.cluster.local` FQDN へのトラフィックがサービス `default/bookstore-v1` と `default/bookstore-v2` に分割される次の例を検討してください。 重みはそれぞれ 90 と 10 です。

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-split
  namespace: default
spec:
  service: bookstore.default.svc.cluster.local
  backends:
  - service: bookstore-v1
    weight: 90
  - service: bookstore-v2
    weight: 10
```

「TrafficSplit」リソースを正しく構成するには、次の条件が満たされていることを確認することが重要です。

- `metadata.namespace` は [namespace added to the mesh](/docs/guides/app_onboarding/namespaces/)
- `metadata.namespace`、`spec.service`、`spec.backend` はすべて同じ名前空間に属します
- `spec.service` は [FQDN of a Kubernetes service](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#services) を指定します 
- `spec.service` と `spec.backends` は Kubernetes サービス オブジェクトに対応します
- すべてのバックエンドの重みの合計は 0 より大きくなければならず、各バックエンドは正の重みを持つ必要があります

`TrafficSplit` リソースが作成されると、osm-edge はクライアント サイドカーの設定を適用して、指定された重みに基づいてルート サービス (`spec.service`) に向けられたトラフィックをバックエンド (`spec.backends`) に分割します。 HTTP トラフィックの場合、リクエストの「Host/Authority」ヘッダーは、「TrafficSplit」リソースで指定されたルート サービスの FQDN と一致する必要があります。 上記の例では、クライアントによって発信された HTTP 要求の「Host/Authority」ヘッダーが、トラフィック分割が機能するために「default/bookstore」サービスの Kubernetes サービス FQDN と一致する必要があることを意味します。
> 注: osm-edge は、元の HTTP リクエストの「Host/Authority」ヘッダーの書き換えを構成しないため、「TrafficSplit」リソースで参照されるバックエンド サービスが、元の HTTP 「Host/Authority」ヘッダーを持つリクエストを受け入れる必要があります。

「TrafficSplit」リソースはサービスへのトラフィック分割のみを構成し、アプリケーションに相互通信の許可を与えないことに注意することが重要です。 したがって、有効な [TrafficTarget](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md#traffictarget) リソースを、 必要に応じてアプリケーション間のトラフィック フローを実現するための「TrafficSplit」構成。

詳細については、[Canary rollouts using SMI Traffic Split](/docs/demos/canary_rollout) のデモを参照してください。