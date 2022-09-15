---
title: 「osm-edge CLI をインストールする」
description: 「このセクションでは、`osm` CLI のインストールと使用について説明します。」
type: docs
weight: 1
---

## 前提条件

- Kubernetes {{< param min_k8s_version >}} 以上を実行している Kubernetes クラスター

## osm-edge CLI をセットアップする

### バイナリリリースから

[リリースページ](https://github.com/flomesh-io/osm-edge/releases)からプラットフォーム固有の圧縮パッケージをダウンロードします。
`osm` バイナリを解凍し、それを `$PATH` に追加して開始します。

#### Linux と macOS

Linux/macOS 上の bash ベースのシェルまたは [Linux 用 Windows サブシステム](https://docs.microsoft.com/windows/wsl/about) では、`curl` を使用して osm-edge リリースをダウンロードし、 「tar」は次のとおりです。

```console
# Specify the osm-edge version that will be leveraged throughout these instructions
OSM_VERSION={{< param osm_edge_version >}}

# Linux curl command only
curl -sL "https://github.com/flomesh-io/osm-edge/releases/download/$OSM_VERSION/osm-$OSM_VERSION-linux-amd64.tar.gz" | tar -vxzf -

# macOS curl command only
curl -sL "https://github.com/flomesh-io/osm-edge/releases/download/$OSM_VERSION/osm-$OSM_VERSION-darwin-amd64.tar.gz" | tar -vxzf -
```

「osm」クライアント バイナリはクライアント マシンで実行され、Kubernetes クラスターで osm-edge を管理できます。 次のコマンドを使用して、Linux または [Windows Subsystem for Linux](https://docs.microsoft.com/windows/wsl/about) の bash ベースのシェルに osm-edge `osm` クライアント バイナリをインストールします。 これらのコマンドは、`osm` クライアント バイナリを `PATH` の標準的なユーザー プログラムの場所にコピーします。

```console
sudo mv ./linux-amd64/osm /usr/local/bin/osm
```

**macOS** の場合、次のコマンドを使用します。

```console
sudo mv ./darwin-amd64/osm /usr/local/bin/osm
```

次のコマンドを使用して、「osm」クライアント ライブラリがパスとそのバージョン番号に正しく追加されたことを確認できます。

```console
osm version
```
### ソースから (Linux、MacOS)

ソースから osm-edge をビルドするには、より多くの手順が必要ですが、最新の変更をテストする最良の方法であり、開発環境で役立ちます。

[Go](https://golang.org/doc/install) 環境が動作している必要があります。

```console
$ git clone git@github.com:flomesh-io/osm-edge.git
$ cd osm
$ make build-osm
```

`make build-osm` は必要な依存関係を取得し、`osm` をコンパイルして `bin/osm` に配置します。 `$PATH` に `bin/osm` を追加して、`osm` を簡単に使用できるようにします。

## osm-edge をインストールする

### osm-edge 構成

デフォルトでは、コントロール プレーン コンポーネントは「osm-system」と呼ばれる Kubernetes 名前空間にインストールされ、コントロール プレーンには、デフォルトで「osm」に設定された一意の識別子属性「mesh-name」が与えられます。
インストール中に、`osm` CLI を使用する場合はフラグを介して、または `helm` CLI を使用する場合は値ファイルを編集することで、ネームスペースとメッシュ名を設定できます。

`mesh-name` は、メッシュ インスタンスを識別および管理するために、インストール中に osm-controller インスタンスに割り当てられる一意の識別子です。

`mesh-name` は、[RFC 1123](https://tools.ietf.org/html/rfc1123) DNS ラベルの制約に従う必要があります。 `mesh-name` は次の条件を満たさなければなりません:

- 最大 63 文字を含む
- 小文字の英数字または「-」のみを含む
- 英数字で始まる
- 英数字で終わる

### osm-edge CLI を使用する

「osm」CLI を使用して、osm-edge コントロール プレーンを Kubernetes クラスターにインストールします。

`osm install` を実行します。

```console
# Install osm control plane components
$ osm install
osm-edge installed successfully in namespace [osm-system] with mesh name [osm]
```

その他のオプションについては、「osm install --help」を実行してください。
