---
title: "osm-edgeをセットアップする"
description: "osm-edge CLI を使用して osm-edge コントロール プレーンをインストールする"
type: ドキュメント
weight: 1
---

# osm-edgeをセットアップする

## 前提条件
この osm-edge {{< param osm_edge_version >}} のデモには以下が必要だ:
  - Kubernetes {{< param min_k8s_version >}} またはその以上を実行するクラスター (using a cloud provider of choice, [minikube](https://minikube.sigs.k8s.io/docs/start/), or similar)
  - [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell))スクリプトを実行できるワークステーション
  - [The Kubernetes command-line tool](https://kubernetes.io/docs/tasks/tools/#kubectl) - `kubectl`
  - ローカルで利用可能な[osm-edge code repo](https://github.com/flomesh-io/osm-edge/) 

> 注意: このドキュメントでは、~/.kube/config に Kubernetes クラスターの資格情報が既にインストールされており、kubectl cluster-info が正常に実行されることを前提としている。



## osm-edge コマンドライン ツールをダウンロードしてインストールする

「osm」コマンドラインツールには、Open Service Mesh のインストールと設定に必要なものがすべて含まれている。バイナリは [osm-edge GitHub releases page](https://github.com/flomesh-io/osm-edge/releases/)で入手できる。
### GNU/Linux

osm-edge {{< param osm_edge_version >}}の64ビットGNU/LinuxまたはmacOSバイナリをダウンロードする。

```bash
system=$(uname -s)
release={{< param osm_edge_version >}}
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-${release}-${system}-amd64.tar.gz | tar -vxzf -
./${system}-amd64/osm version
```

### macOS

osm-edge {{< param osm_edge_version >}}用の64ビットmacOSバイナリをダウンロードする。

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(uname -m)
release={{< param osm_edge_version >}}
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
. /${system}-${arch}/osm version
```

osm CLI は、 [this guide](/docs/guides/cli)に従ってソースからコンパイルできる。

## Kubernetes に osm-edge をインストールする

「osm」バイナリをダウンロードして解凍したら、Open Service Mesh を Kubernetes クラスターにインストールする準備が整った。

以下のコマンドは、Kubernetes クラスターに osm-edge をインストールする方法を示す。このコマンドは、[Prometheus](https://github.com/prometheus/prometheus)、[Grafana](https://github.com/grafana/grafana)、および [Jaeger](https://github.com/jaegertracing/jaeger) の統合を有効にする。`value.yaml` ファイルにある `osm.enablePermissiveTrafficPolicy` チャートパラメータは、osm-edge に対して、ポリシーを無視し、トラフィックがポッド間を自由に流れるように指示する。Permissive Traffic Policy モードを有効にすると、新しいポッドはEnvoyで注入されるが、トラフィックはプロキシを経由して流れ、アクセス制御ポリシーによってブロックされることはない。

> 注意: Permissive Traffic Policy モードは、SMI ポリシーの作成に時間がかかるブラウンフィールド展開にとって重要な機能だ。オペレーターが SMI ポリシーを設計している間、既存のサービスは osm-edge がインストールされる前と同様に動作し続ける。

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge will be installed
export osm_mesh_name=osm # Replace osm with the desired osm-edge mesh name

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.deployPrometheus=true \
    --set=osm.deployGrafana=true \
    --set=osm.deployJaeger=true
```

osm-edge の Prometheus、Grafana、および Jaeger との統合の詳細については、[observability documentation](/docs/guides/observability/)を参照してください。

## 次のステップ

osm-edge コントロールプレーンが起動して実行されるようになったので、 [add applications](/docs/getting_started/install_apps/)をメッシュに追加する。