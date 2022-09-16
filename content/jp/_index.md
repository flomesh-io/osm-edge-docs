---
title: "osm-edgeドキュメント"
description: ""
type: docs

---

シンプルで整ったスタンドアロンサービスメッシュ。

osm-edge は[Kubernetes](https://kubernetes.io/)上で実行される。osm-edge コントロールプレーンは [Pipy Repo](https://flomesh.io/docs/en/operating/repo/0-intro)を実装し、[SMI](https://smi-spec.io/)  API で設定される。osm-edge は、アプリケーションの各インスタンスの横にサイドカーコンテナーとして Pipy プロキシを挿入する。

osm-edge について詳しく知るには:
*  [getting started articles](/docs/getting_started/)を一通り読んで、osm-edge をインストールし、サンプルアプリケーションを実行してください。
* [overview of osm-edge](/docs/overview/about/)と、 [the design of its components](/docs/overview/osm_components/)の詳細をお読みください。
*　[デモ](/docs/demos/) セクションから追加のサンプル シナリオを実行する。