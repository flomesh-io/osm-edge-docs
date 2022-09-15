---
title: 「ヘルス プローブの構成」
description: "osm-edge がアプリケーションのヘルス プローブを処理する方法と、失敗した場合の対処方法"
Alias: "/docs/application_health_probes"
type: docs
---

# ヘルス プローブの構成

## 概要

[ヘルス プローブ](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) をアプリケーションに実装することは、Kubernetes がいくつかのタスクを自動化して改善する優れた方法です。 エラーが発生した場合の可用性。

osm-edge はアプリケーション Pod を再構成して、プロキシ サイドカーを介してすべての受信および送信ネットワーク トラフィックをリダイレクトするため、kubelet によって呼び出される「httpGet」および「tcpSocket」ヘルス プローブは、プロキシに必要な mTLS コンテキストがないために失敗します。

「httpGet」ヘルス プローブがメッシュ内から期待どおりに機能し続けるために、osm-edge は、プロキシを介してプローブ エンドポイントを公開するための構成を追加し、プロキシに公開されたエンドポイントを参照するように新しい Pod のプローブ定義を書き換えます。 元のプローブのすべての機能が引き続き使用されます。osm-edge は、kubelet がプローブと通信できるように、単にプロキシを前面に配置します。

メッシュで「tcpSocket」ヘルス プローブをサポートするには、特別な構成が必要です。 osm-edge はすべてのネットワーク トラフィックを Pipy 経由でリダイレクトするため、Pod ではすべてのポートが開いているように見えます。 これにより、Pipy サイドカーが挿入された Pod にルーティングされたすべての TCP 接続が成功したように見えます。 「tcpSocket」ヘルス プローブがメッシュで期待どおりに機能するように、osm-edge はプローブを「httpGet」プローブに書き換え、「osm-healthcheck」公開エンドポイントで Pipy プロキシをバイパスする「iptables」コマンドを追加します。 「osm-healthcheck」コンテナーが Pod に追加され、kubelet からの HTTP ヘルス プローブ リクエストを処理します。 ハンドラーは、リクエストの「Original-Tcp-port」ヘッダーから元の TCP ポートを取得し、指定されたポートでソケットを開こうとします。 TCP 接続が成功したかどうかは、`httpGet` プローブの応答ステータス コードに反映されます。

| 調査       | 道                 | ポート  |
| ----------- | -------------------- | ----- |
| 活気    | /osm-liveness-probe  | 15901 |
| 準備   | /osm-readiness-probe | 15902 |
| 起動     | /osm-startup-probe   | 15903 |
| 健康診断 | /osm-healthcheck     | 15904 |

HTTP および「tcpSocket」プローブの場合、ポートとパスが変更されます。 HTTPS プローブの場合、ポートは変更されますが、パスは変更されません。

定義済みの「httpGet」および「tcpSocket」プローブのみが変更されます。 プローブが定義されていない場合、プローブはその場所に追加されません。 `exec` プローブ (`grpc_health_probe` を使用するものを含む) は決して変更されず、コマンドが `localhost` 外部のネットワーク アクセスを必要としない限り、期待どおりに機能し続けます。

## 例

次の例は、osm-edge がメッシュ内の Pod のヘルス プローブを処理する方法を示しています。

### HTTP

次の `livenessProbe` を使用してコンテナーを定義する Pod 仕様を検討してください。

```yaml
livenessProbe:
  httpGet:
    path: /liveness
    port: 14001
    scheme: HTTP
```

Pod が作成されると、osm-edge はプローブを次のように変更します。

```yaml
livenessProbe:
  httpGet:
    path: /osm-liveness-probe
    port: 15901
    scheme: HTTP
```

Pod のプロキシには、次の Pipy 構成が含まれます。

元のプローブ ポート 14001 にマップされる Pipy クラスター:

```json
{
  "Probes": {
      "ReadinessProbes": null,
      "LivenessProbes": [
        {
          "httpGet": {
            "path": "/osm-liveness-probe",
            "port": 15901,
            "scheme": "HTTP"
          },
          "timeoutSeconds": 1,
          "periodSeconds": 10,
          "successThreshold": 1,
          "failureThreshold": 3
        }
      ],
      "StartupProbes": null
    }
  }
}
```

上記のクラスターにマッピングされたポート 15901 の `/osm-liveness-probe` にある新しいプロキシ公開 HTTP エンドポイントのリスナー:

```js
.listen(probeScheme ? 15901 : 0)
.link(
  'http_liveness', () => probeScheme === 'HTTP',
  'connection_liveness', () => Boolean(probeTarget),
  'deny_liveness'
)
```

### `tcpソケット`

次の `livenessProbe` を使用してコンテナーを定義する Pod 仕様を検討してください。

```yaml
livenessProbe:
  tcpSocket:
    port: 14001
```

Pod が作成されると、osm-edge はプローブを次のように変更します。

```yaml
livenessProbe:
  httpGet:
    httpHeaders:
    - name: Original-Tcp-Port
      value: "14001"
    path: /osm-healthcheck
    port: 15904
    scheme: HTTP
```

ポート 15904 へのリクエストは、Pipy プロキシをバイパスし、「osm-healthcheck」エンドポイントに転送されます。

## メッシュ内のポッドの健全性を確認する方法

Kubernetes は、起動、活性、および準備プローブで構成された Pod の正常性エンドポイントを自動的にポーリングします。

起動プローブが失敗すると、Kubernetes はイベントを生成し (「kubectl describe pod <pod name>」で表示)、Pod を再起動します。 `kubectl describe` の出力は次のようになります。

```console
...
Events:
  Type     Reason     Age              From               Message
  ----     ------     ----             ----               -------
  Normal   Scheduled  17s              default-scheduler  Successfully assigned bookstore/bookstore-v1-699c79b9dc-5g8zn to osm-control-plane
  Normal   Pulled     16s              kubelet            Successfully pulled image "openservicemesh/init:v0.8.0" in 26.5835ms
  Normal   Created    16s              kubelet            Created container osm-init
  Normal   Started    16s              kubelet            Started container osm-init
  Normal   Pulling    16s              kubelet            Pulling image "openservicemesh/init:v0.8.0"
  Normal   Pulling    15s              kubelet            Pulling image "flomesh/pipy:0.5.0"
  Normal   Pulling    15s              kubelet            Pulling image "openservicemesh/bookstore:v0.8.0"
  Normal   Pulled     15s              kubelet            Successfully pulled image "openservicemesh/bookstore:v0.8.0" in 319.9863ms
  Normal   Started    15s              kubelet            Started container bookstore-v1
  Normal   Created    15s              kubelet            Created container bookstore-v1
  Normal   Pulled     14s              kubelet            Successfully pulled image "flomesh/pipy:0.5.0" in 755.2666ms
  Normal   Created    14s              kubelet            Created container pipy
  Normal   Started    14s              kubelet            Started container pipy
  Warning  Unhealthy  13s              kubelet            Startup probe failed: Get "http://10.244.0.23:15903/osm-startup-probe": dial tcp 10.244.0.23:15903: connect: connection refused
  Warning  Unhealthy  3s (x2 over 8s)  kubelet            Startup probe failed: HTTP probe failed with statuscode: 503
```

liveness プローブが失敗すると、Kubernetes はイベントを生成し (「kubectl describe pod <pod name>」で表示)、Pod を再起動します。 `kubectl describe` の出力は次のようになります。

```console
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  59s                default-scheduler  Successfully assigned bookstore/bookstore-v1-746977967c-jqjt4 to osm-control-plane
  Normal   Pulling    58s                kubelet            Pulling image "openservicemesh/init:v0.8.0"
  Normal   Created    58s                kubelet            Created container osm-init
  Normal   Started    58s                kubelet            Started container osm-init
  Normal   Pulled     58s                kubelet            Successfully pulled image "openservicemesh/init:v0.8.0" in 23.415ms
  Normal   Pulled     57s                kubelet            Successfully pulled image "flomesh/pipy:0.5.0" in 678.1391ms
  Normal   Pulled     57s                kubelet            Successfully pulled image "openservicemesh/bookstore:v0.8.0" in 230.3681ms
  Normal   Created    57s                kubelet            Created container pipy
  Normal   Pulling    57s                kubelet            Pulling image "flomesh/pipy:0.5.0"
  Normal   Started    56s                kubelet            Started container pipy
  Normal   Pulled     44s                kubelet            Successfully pulled image "openservicemesh/bookstore:v0.8.0" in 20.6731ms
  Normal   Created    44s (x2 over 57s)  kubelet            Created container bookstore-v1
  Normal   Started    43s (x2 over 57s)  kubelet            Started container bookstore-v1
  Normal   Pulling    32s (x3 over 58s)  kubelet            Pulling image "openservicemesh/bookstore:v0.8.0"
  Warning  Unhealthy  32s (x6 over 50s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    32s (x2 over 44s)  kubelet            Container bookstore-v1 failed liveness probe, will be restarted
```

readiness プローブが失敗すると、Kubernetes はイベント (「kubectl describe pod <pod name>」で表示) を生成し、Pod がサポートしている可能性のあるサービス宛てのトラフィックが異常な Pod にルーティングされないようにします。 readiness プローブが失敗した Pod の「kubectl describe」出力は、次のようになります。

```console
...
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  32s               default-scheduler  Successfully assigned bookstore/bookstore-v1-5848999cb6-hp6qg to osm-control-plane
  Normal   Pulling    31s               kubelet            Pulling image "openservicemesh/init:v0.8.0"
  Normal   Pulled     31s               kubelet            Successfully pulled image "openservicemesh/init:v0.8.0" in 19.8726ms
  Normal   Created    31s               kubelet            Created container osm-init
  Normal   Started    31s               kubelet            Started container osm-init
  Normal   Created    30s               kubelet            Created container bookstore-v1
  Normal   Pulled     30s               kubelet            Successfully pulled image "openservicemesh/bookstore:v0.8.0" in 314.3628ms
  Normal   Pulling    30s               kubelet            Pulling image "openservicemesh/bookstore:v0.8.0"
  Normal   Started    30s               kubelet            Started container bookstore-v1
  Normal   Pulling    30s               kubelet            Pulling image "flomesh/pipy:0.5.0"
  Normal   Pulled     29s               kubelet            Successfully pulled image "flomesh/pipy:0.5.0" in 739.3931ms
  Normal   Created    29s               kubelet            Created container pipy
  Normal   Started    29s               kubelet            Started container pipy
  Warning  Unhealthy  0s (x3 over 20s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 503
```

Pod の `status` は、`kubectl get pod` 出力に表示される準備ができていないことも示します。 例えば: 
```console
NAME                            READY   STATUS    RESTARTS   AGE
bookstore-v1-5848999cb6-hp6qg   1/2     Running   0          85s
```

Pod のヘルス プローブは、Pod の必要なポートを転送し、「curl」またはその他の HTTP クライアントを使用してリクエストを発行することにより、手動で呼び出すこともできます。 たとえば、bookstore-v1 デモ Pod の liveness プローブを確認するには、ポート 15901 を転送します。

```bash
kubectl port-forward -n bookstore deployment/bookstore-v1 15901
```

次に、別の端末インスタンスで、「curl」を使用してエンドポイントを確認できます。 次の例は、健全な bookstore-v1 を示しています。

```console
$ curl -i localhost:15901/osm-liveness-probe
HTTP/1.1 200 OK
date: Wed, 31 Mar 2021 16:00:01 GMT
content-length: 1396
content-type: text/html; charset=utf-8
x-pipy-upstream-service-time: 1
server: pipy

<!doctype html>
<html itemscope="" itemtype="http://schema.org/WebPage" lang="en">
  ...
</html>
```

## 既知の問題点

- [#3773](https://github.com/openservicemesh/osm/issues/3773)

## トラブルシューティング

いずれかの正常性プローブが一貫して失敗する場合は、次の手順を実行して根本原因を特定します。

1. メッシュ内の Pod の「httpGet」および「tcpSocket」プローブが変更されていることを確認します。

    メッシュ内で機能し続けるには、起動、活性、および準備の `httpGet` プローブを osm-edge で変更する必要があります。 liveness、readiness、および startup `httpGet` プローブ用に、ポートをそれぞれ 15901、15902、および 15903 に変更する必要があります。 HTTP (HTTPS ではない) プローブのみが、`/osm-liveness-probe`、`/osm-readiness-probe`、または `/osm-startup-probe` に加えてパスが変更されます。

  また、Pod の Pipy 構成に、変更されたエンドポイントのリスナーが含まれていることを確認します。

    `tcpSocket` プローブがメッシュで機能するには、`httpGet` プローブに書き換える必要があります。 liveness、readiness、および startup プローブのために、ポートを 15904 に変更する必要があります。 パスは `/osm-healthcheck` に設定する必要があります。 HTTP ヘッダー「Original-Tcp-Port」は、「tcpSocket」プローブ定義で指定された元のポートに設定する必要があります。 また、「osm-healthcheck」コンテナーが実行されていることを確認します。 詳細については、「osm-healthcheck」ログを調べてください。

    詳細については、[上記の例](#examples) を参照してください。

2. Pod のスケジューリングまたは開始中に、Kubernetes で他のエラーが発生したかどうかを確認します。

    異常な Pod の「kubectl describe」で最近発生した可能性のあるエラーを探します。 エラーを解決し、Pod の正常性を再度確認します。

3. Pod で実行時エラーが発生したかどうかを確認します。

    `kubectl logs` でログを調べて、コンテナの起動後に発生した可能性のあるエラーを探します。 エラーを解決し、Pod の正常性を再度確認します。
