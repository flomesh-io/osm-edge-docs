---
title: 「iptables リダイレクトのトラブルシューティング」
description: 「iptablesリダイレクトトラブルシューティングガイド」
type: docs
---

## トラフィックのリダイレクトが期待どおりに機能しない場合

### 1. Pod に Pipy サイドカー コンテナーが挿入されていることを確認する

トラフィックのリダイレクトが期待どおりに機能するには、アプリケーション ポッドに Pipy プロキシ サイドカーを挿入する必要があります。 アプリケーション ポッドが実行中で、Pipy プロキシ サイドカー コンテナーが準備完了状態であることを確認して、これを確認します。

```console
$ kubectl get pod test-58d4f8ff58-wtz4f -n test
NAME                                READY   STATUS    RESTARTS   AGE
test-58d4f8ff58-wtz4f               2/2     Running   0          32s
```

### 2. osm-edge の init コンテナが正常に実行を終了したことを確認します

osm-edge の init コンテナ `osm-init` は、Pipy プロキシ サイドカーを介してアプリケーション トラフィックをプロキシするトラフィック リダイレクション ルールを使用して、サービス メッシュ内の個々のアプリケーション ポッドを初期化する役割を担います。 トラフィック リダイレクト ルールは、ポッド内のアプリケーション コンテナが実行される前に実行される一連の「iptables」コマンドを使用して設定されます。

アプリケーション ポッドで「kubectl describe」を実行し、「osm-init」コンテナが終了コード 0 で終了したことを確認して、osm-edge の init コンテナの実行が正常に完了したことを確認します。コンテナの「State」プロパティは、この情報を提供します。

```console
$ kubectl describe pod test-58d4f8ff58-wtz4f -n test
Name:         test-58d4f8ff58-wtz4f
Namespace:    test
...
...
Init Containers:
  osm-init:
    Container ID:  containerd://98840f655f2310b2f441e11efe9dfcf894e4c57e4e26b928542ee698159100c0
    Image:         openservicemesh/init:2c18593efc7a31986a6ae7f412e73b6067e11a57
    Image ID:      docker.io/openservicemesh/init@sha256:24456a8391bce5d254d5a1d557d0c5e50feee96a48a9fe4c622036f4ab2eaf8e
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      iptables -t nat -N PROXY_INBOUND && iptables -t nat -N PROXY_IN_REDIRECT && iptables -t nat -N PROXY_OUTPUT && iptables -t nat -N PROXY_REDIRECT && iptables -t nat -A PROXY_REDIRECT -p tcp -j REDIRECT --to-port 15001 && iptables -t nat -A PROXY_REDIRECT -p tcp --dport 15000 -j ACCEPT && iptables -t nat -A OUTPUT -p tcp -j PROXY_OUTPUT && iptables -t nat -A PROXY_OUTPUT -m owner --uid-owner 1500 -j RETURN && iptables -t nat -A PROXY_OUTPUT -d 127.0.0.1/32 -j RETURN && iptables -t nat -A PROXY_OUTPUT -j PROXY_REDIRECT && iptables -t nat -A PROXY_IN_REDIRECT -p tcp -j REDIRECT --to-port 15003 && iptables -t nat -A PREROUTING -p tcp -j PROXY_INBOUND && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15010 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15901 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15902 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15903 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp -j PROXY_IN_REDIRECT
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 22 Mar 2021 09:26:14 -0700
      Finished:     Mon, 22 Mar 2021 09:26:14 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from frontend-token-5g488 (ro)
```

## 送信 IP 範囲の除外が構成されている場合

デフォルトでは、基礎となるトランスポート プロトコルとして TCP を使用するすべてのトラフィックは、Pipy プロキシ サイドカー コンテナー経由でリダイレクトされます。 これは、アプリケーションからのすべての TCP ベースのアウトバウンド トラフィックが、サービス メッシュ ポリシーに基づいて Pipy プロキシ サイドカー経由でリダイレクトおよびルーティングされることを意味します。 アウトバウンド IP 範囲の除外が構成されている場合、これらの IP 範囲に属するトラフィックは Pipy サイドカーにプロキシされません。

アウトバウンド IP 範囲が除外されるように構成されているが、サービス メッシュ ポリシーの対象である場合は、期待どおりに構成されていることを確認します。

### 1. `osm-mesh-config` MeshConfig リソースでアウトバウンド IP 範囲が正しく構成されていることを確認します

除外する送信 IP 範囲が正しく設定されていることを確認します。

```console
# Assumes osm-edge is installed in the osm-system namespace
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.outboundIPRangeExclusionList}{"\n"}'
["1.1.1.1/32","2.2.2.2/24"]
```

出力には、送信トラフィックのリダイレクトから除外される IP 範囲が表示されます (上記の例では「["1.1.1.1/32","2.2.2.2/24"]」)。

### 2. アウトバウンド IP 範囲が init コンテナー仕様に含まれていることを確認する

アウトバウンド IP 範囲の除外が構成されている場合、osm-edge の「osm-injector」サービスは、「osm-mesh-config」「MeshConfig」リソースからこの構成を読み取り、これらの範囲に対応する「iptables」ルールをプログラムして、アウトバウンドから除外されるようにします。 Pipy サイドカー プロキシ経由のトラフィック リダイレクト。

osm-edge の「osm-init」init コンテナー仕様に、除外する構成済みの送信 IP 範囲に対応するルールがあることを確認します。

```console
$ kubectl describe pod test-58d4f8ff58-wtz4f -n test
Name:         test-58d4f8ff58-wtz4f
Namespace:    test
...
...
Init Containers:
  osm-init:
    Container ID:  containerd://98840f655f2310b2f441e11efe9dfcf894e4c57e4e26b928542ee698159100c0
    Image:         openservicemesh/init:2c18593efc7a31986a6ae7f412e73b6067e11a57
    Image ID:      docker.io/openservicemesh/init@sha256:24456a8391bce5d254d5a1d557d0c5e50feee96a48a9fe4c622036f4ab2eaf8e
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      iptables -t nat -N PROXY_INBOUND && iptables -t nat -N PROXY_IN_REDIRECT && iptables -t nat -N PROXY_OUTPUT && iptables -t nat -N PROXY_REDIRECT && iptables -t nat -A PROXY_REDIRECT -p tcp -j REDIRECT --to-port 15001 && iptables -t nat -A PROXY_REDIRECT -p tcp --dport 15000 -j ACCEPT && iptables -t nat -A OUTPUT -p tcp -j PROXY_OUTPUT && iptables -t nat -A PROXY_OUTPUT -m owner --uid-owner 1500 -j RETURN && iptables -t nat -A PROXY_OUTPUT -d 127.0.0.1/32 -j RETURN && iptables -t nat -A PROXY_OUTPUT -j PROXY_REDIRECT && iptables -t nat -A PROXY_IN_REDIRECT -p tcp -j REDIRECT --to-port 15003 && iptables -t nat -A PREROUTING -p tcp -j PROXY_INBOUND && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15010 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15901 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15902 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15903 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp -j PROXY_IN_REDIRECT && iptables -t nat -I PROXY_OUTPUT -d 1.1.1.1/32 -j RETURN && && iptables -t nat -I PROXY_OUTPUT -d 2.2.2.2/24 -j RETURN
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 22 Mar 2021 09:26:14 -0700
      Finished:     Mon, 22 Mar 2021 09:26:14 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from frontend-token-5g488 (ro)
```

上記の例では、次の「iptables」コマンドは、構成された送信 IP 範囲 (「1.1.1.1/32 および 2.2.2.2/24」) が Pipy プロキシ サイドカーにリダイレクトされるのを明示的に無視する役割を果たします。
```console
iptables -t nat -I PROXY_OUTPUT -d 1.1.1.1/32 -j RETURN
iptables -t nat -I PROXY_OUTPUT -d 2.2.2.2/24 -j RETURN
```

## 送信ポートの除外が構成されている場合

デフォルトでは、基礎となるトランスポート プロトコルとして TCP を使用するすべてのトラフィックは、Pipy プロキシ サイドカー コンテナー経由でリダイレクトされます。 これは、アプリケーションからのすべての TCP ベースのアウトバウンド トラフィックが、サービス メッシュ ポリシーに基づいて Pipy プロキシ サイドカー経由でリダイレクトおよびルーティングされることを意味します。 送信ポートの除外が構成されている場合、これらのポートに属するトラフィックは Pipy サイドカーにプロキシされません。

アウトバウンド ポートが除外されるように構成されているが、サービス メッシュ ポリシーの対象である場合は、期待どおりに構成されていることを確認します。

### 1. `osm-mesh-config` MeshConfig リソースでグローバル アウトバウンド ポートが正しく構成されていることを確認します

除外する送信ポートが正しく設定されていることを確認します。
```console
# Assumes osm-edge is installed in the osm-system namespace
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.outboundPortExclusionList}{"\n"}'
[6379,7070]
```

出力には、アウトバウンド トラフィックのリダイレクトから除外されるポートが表示されます (上記の例では「[6379,7070]」)。

### 2. Pod レベルの送信ポートが Pod で正しく注釈されていることを確認する

ポッドで除外するアウトバウンド ポートが正しく設定されていることを確認します。

```console
$ kubectl get pod POD_NAME -o jsonpath='{.metadata.annotations}' -n POD_NAMESPACE'
map[openservicemesh.io/outbound-port-exclusion-list:8080]
```

出力には、ポッドのアウトバウンド トラフィック リダイレクトから除外されるポートが表示されます (上記の例では「8080」)。

### 3. アウトバウンド ポートが init コンテナー仕様に含まれていることを確認する

送信ポートの除外が構成されている場合、osm-edge の「osm-injector」サービスは、「osm-mesh-config」「MeshConfig」リソースとポッドの注釈からこの構成を読み取り、これらの範囲に対応する「iptables」ルールをプログラムします。 Pipy サイドカー プロキシを介したアウトバウンド トラフィックのリダイレクトから除外されるようにします。

osm-edge の `osm-init` init コンテナー仕様に、除外する構成済みの送信ポートに対応するルールがあることを確認します。

```console
$ kubectl describe pod test-58d4f8ff58-wtz4f -n test
Name:         test-58d4f8ff58-wtz4f
Namespace:    test
...
...
Init Containers:
  osm-init:
    Container ID:  containerd://98840f655f2310b2f441e11efe9dfcf894e4c57e4e26b928542ee698159100c0
    Image:         openservicemesh/init:2c18593efc7a31986a6ae7f412e73b6067e11a57
    Image ID:      docker.io/openservicemesh/init@sha256:24456a8391bce5d254d5a1d557d0c5e50feee96a48a9fe4c622036f4ab2eaf8e
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      iptables -t nat -N PROXY_INBOUND && iptables -t nat -N PROXY_IN_REDIRECT && iptables -t nat -N PROXY_OUTPUT && iptables -t nat -N PROXY_REDIRECT && iptables -t nat -A PROXY_REDIRECT -p tcp -j REDIRECT --to-port 15001 && iptables -t nat -A PROXY_REDIRECT -p tcp --dport 15000 -j ACCEPT && iptables -t nat -A OUTPUT -p tcp -j PROXY_OUTPUT && iptables -t nat -A PROXY_OUTPUT -m owner --uid-owner 1500 -j RETURN && iptables -t nat -A PROXY_OUTPUT -d 127.0.0.1/32 -j RETURN && iptables -t nat -A PROXY_OUTPUT -j PROXY_REDIRECT && iptables -t nat -A PROXY_IN_REDIRECT -p tcp -j REDIRECT --to-port 15003 && iptables -t nat -A PREROUTING -p tcp -j PROXY_INBOUND && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15010 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15901 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15902 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15903 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp -j PROXY_IN_REDIRECT && iptables -t nat -I PROXY_OUTPUT -p tcp --match multiport --dports 6379,7070,8080 -j RETURN
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 22 Mar 2021 09:26:14 -0700
      Finished:     Mon, 22 Mar 2021 09:26:14 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from frontend-token-5g488 (ro)
```

上記の例では、次の「iptables」コマンドは、構成された送信ポート (「6379、7070、および 8080」) が Pipy プロキシ サイドカーにリダイレクトされないように明示的に無視する役割を果たします。
```console
iptables -t nat -I PROXY_OUTPUT -p tcp --match multiport --dports 6379,7070,8080 -j RETURN
```