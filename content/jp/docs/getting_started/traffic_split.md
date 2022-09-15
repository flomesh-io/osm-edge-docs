---
title: "トラフィックスプリットの設定"
description: "SMI Traffic Split APIによるサービス間のトラフィックバランス調整"
type: ドキュメント
weight: 4
---

# 2 つのサービス間のトラフィックスプリットの設定

ここでは、一般にトラフィックスプリットとして知られている 2 つの Kubernetes サービス間のトラフィックをスプリットする方法を示す。バックエンドの「bookstore」サービスと「bookstore-v2」サービスの間で、ルートの「bookstore」サービスに向けられたトラフィックをスプリットする。

### bookstore v2 アプリケーションをデプロイする

SMIトラフィックアクセスとスプリットポリシーの使用法を示すために、バージョン v2 のブックストアアプリケーション (bookstore-v2) をデプロイする。- openshift を使用している場合は、[installation guide](/docs/install/#openshift)で指定されているように、bookstore-v2 サービス アカウントにセキュリティコンテキスト制約を追加する必要があることに注意してください。

```bash
# Contains the bookstore-v2 Kubernetes Service, Service Account, Deployment and SMI Traffic Target resource to allow
# `bookbuyer` to communicate with `bookstore-v2` pods
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/apps/bookstore-v2.yaml
```

`bookstore-v2` Pod が `bookstore` 名前空間で実行されるのを待。次に、bookstore の v2 にアクセスするために、`./scripts/port-forward-all.sh` スクリプトを終了して再起動する。

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

bookstore-v2 サービスにはまだトラフィックが流れていないため、カウンターは増加しない。

### SMI トラフィックスプリットの作成

SMI トラフィックスプリットポリシーをデプロイして、ルートブックストアサービスに送信されるトラフィックの 100% をブックストア サービス バックエンドに転送する。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v1.yaml
```

_注意: ルートサービスには、任意のKubernetesサービスを指定することができる。ラベルセレクタは一切ない。また、トラフィックスプリットリソースで指定されたバックエンドサービスと重複する必要はない。ルートサービスは、SMI Traffic Splitリソースにおいて、.<namespace>サフィックスの有無にかかわらず、サービス名として参照することができる。_

bookstore-v2 ブラウザ ウィンドウから販売された書籍の数は 0 のままである必要がある。これは、bookbuyer が bookstore サービスにトラフィックを送信しており、bookstore-v2 サービスにリクエストを送信しているアプリケーションがないという事実に加えて、現在のトラフィックスプリットポリシーが bookstore に対して現在 100 に重み付けされているためだ。以下を実行し、**Backends** プロパティを表示することで、トラフィックスプリットポリシーを確認できる。
```bash
kubectl describe trafficsplit bookstore-split -n bookstore
```

### Bookstore v2 へのトラフィックスプリット

SMIトラフィックスプリットポリシーを更新して、ルートブックストアサービスに送信されるトラフィックの50パーセントをブックストアサービスに、50パーセントをブックストア-v2サービスに向けるように、ブックストア-v2バックエンドを仕様に追加し、ウエイトフィールドを変更することによって行う。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/split/traffic-split-50-50.yaml
```

変更が反映されるのを待ち、ブラウザウィンドウで `bookstore` と `bookstore-v2` のカウンターの増加を観察してください。両方のカウンタがインクリメントしているはずだ。

- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

### すべてのトラフィックを Bookstore v2 にスプリット

`bookstore-split` TrafficSplit を更新して、すべてのトラフィックが `bookstore-v2` に向かうように設定する。

```bash
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v2.yaml
```

変更が反映されるのを待ち、「bookstore-v2」のカウンターが増加し、ブラウザウィンドウの「bookstore」がフリーズするのを観察する。

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
- [http://localhost:8083](http://localhost:8084) - **bookstore**

これで、「bookstore」サービスに向けられたすべてのトラフィックが「bookstore-v2」に流れる。

## 次のステップ

- [Configure observability with Prometheus and Grafana](/docs/getting_started/observability/)
- [Cleanup sample applications and uninstall osm-edge](/docs/getting_started/cleanup/)