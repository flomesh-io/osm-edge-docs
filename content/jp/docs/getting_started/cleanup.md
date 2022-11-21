---
title: "osm-edgeをアンインストールする"
description: "osm-edgeとbookstoreアプリケーションをアンインストールする"
type: docs
weight: 6
---

# osm-edgeをアンインストールする
入門セクションの記事では、osm-edge とサンプル アプリケーションのインストールについて概説する。これらのリソースをすべてアンインストールするには、サンプルアプリケーションと関連するSMIリソースを削除し、osm-edge コントロール プレーンとクラスター全体のosm-edgeリソースをアンインストールする。

## サンプルアプリケーションを削除する

サンプルアプリケーションと関連する SMI リソースをクリーンアップするには、それらの名前空間を削除する。例えば、

```bash
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

## osm-edgeコントロールプレーンのアンインストール

osm-edgeコントロールプレーンをアンインストールするには、「osm uninstall mesh」を使用する。

```bash
osm uninstall mesh
```

## sosm-edge クラスター全体のリソースをアンインストールする

osm-edgeクラスター全体のリソースをアンインストールするには、「osm uninstall mesh --delete-cluster-wide-resources」を使用する。

```bash
osm uninstall mesh --delete-cluster-wide-resources
```

osm-edge のアンインストールの詳細については、[uninstallation guide](docs/guides/uninstall/)を参照してください。
