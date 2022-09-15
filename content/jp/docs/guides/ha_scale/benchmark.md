---
title: 「データ プレーンのベンチマーク」
description: 「osm-edge と osm データ プレーンのベンチマーク」
type: docs
weight: 1
---

osm-edge は、リソースに制約のあるエッジ環境でもクラウドで使用されるサービス メッシュ機能を使用できるように、高性能リソースと共にサービス メッシュ機能を提供することを目的としています。

このテストでは、osm-edge (v1.1.0) と osm (v1.1.0) に対してベンチマークが実施されました。 主な焦点は、サービス TPS、2 つの異なるメッシュを使用する場合のレイテンシー分散、およびデータ プレーンのリソース オーバーヘッドの監視です。

osm-edge はデータ プレーンとして Pipy を使用します。 osm は Envoy をデータ プレーンとして使用します。

## テスト環境

ベンチマークは、Tencent Cloud CVM で実行されている Kubernetes クラスターでテストされました。 クラスターには 2 つの標準 S5 ノードがあります。 osm-edge と osm はどちらも **ルース トラフィック モードと mTLS 上であり、その他の設定はデフォルトです**。

* Kubernetes: k3s v1.20.14+k3s2
* OS: Ubuntu 20.04
* ノード: 16c32g * 2
* ロード ジェネレーター: 8c16g

テスト アプリケーションは、共通の SpringCloud マイクロサービス アーキテクチャを使用します。 アプリケーションは、SpringCloud を使用した bookinfo アプリケーションの SpringCloud 実装である [flomesh-bookinfo-demo](https://github.com/flomesh-io/flomesh-bookinfo-demo/) から取得されます。 テストでは、すべてのサービスが使用されるわけではなく、API ゲートウェイと Bookinfo Ratings サービスが選択されます。

![](https://user-images.githubusercontent.com/2224492/178288704-3aa44151-4c57-4538-9a0a-55310bb4f200.png)

サービスへの外部アクセスは Ingress を介して提供されます。 Load Generator は、一般的な Apache Jmeter 5.5 を使用します。

テストでは、2 つのリンクが選択されました。1 つは Ingress を介した Bookinfo 評価サービスへの直接アクセス (以下、評価) であり、もう 1 つは API ゲートウェイ (以下、ゲートウェイ評価) を介して通過します。 これら 2 つのリンクを選択する理由は、単一のサイドカーと複数のサイドアームの両方のシナリオをカバーするためです。

## 手順

非メッシュ (つまり、サイドカー インジェクションなし、以降非メッシュと呼ぶ) でテストを開始し、それぞれ osm-edge と osm メッシュでテストします。

エッジのリソースが制約されたシナリオをシミュレートするために、グリッドを使用するときにサイドカーによって使用される CPU が制限され、1 コアと 2 コアのシナリオがそれぞれテストされます。

したがって、各ラウンドで 2 つの異なるリンクを使用して、5 ラウンドのテストが行われます。

## パフォーマンス

Jmeter はテスト中に 200 のスレッドを使用し、テストを 5 分間実行します。 テストの各ラウンドの前に、2 分間のウォームアップが行われます。

スペースの都合上、サイドカー リソースの制約が異なるゲートウェイ評価リンクの結果のみを以下に示します。

*注: 表のサイドカーは、API Gateway サイドカーを指します*。

| グリッド | サイドカーの最大 CPU | TPS | 90th | 95th | 99th | サイドカーの CPU 使用率 | サイドカーのメモリ使用量 |
|----------|:-----------------|------|:-----|:-----|:-----|:-----------------|:----------------|
| 非メッシュ | NA | 3283 | 87 | 89 | 97 | NA | NA | NA |
| osm-edge | 2 | 3395 | 77 | 79 | 84 | 130% | 52 |
| osm | 2 | 2189 | 102 | 104 | 109 | 200% | 108 |
| osm-edge | 1 | 2839 | 76 | 77 | 79 | 100% | 34 |
| The osm-edge | 1 | 1097 | 201 | 203 | 285 | 100% | 105 |


### サイドカー 2 コア

サイドカー 2 コア シナリオでは、osm-edge グリッドを使用すると、グリッドを使用しない場合よりも TPS がわずかに改善され、レイテンシも改善されます。 API Gateway と Bookinfo Ratings の両方のサイドカーは、Bookinfo Ratings サービス自体がパフォーマンスの限界に達しているときに、まだ 2 つのコア (わずか 65%) を使い果たしていません。

OSM グリッドの TPS は 30% 近く低下し、API ゲートウェイ サイドカー CPU はフル稼働しており、これがボトルネックでした。

メモリに関しては、osm-edge と osm sidecar のメモリ使用量はそれぞれ 50 MiB と 105 MiB です。

**TPS**

![](https://user-images.githubusercontent.com/2224492/178294418-d3d63aef-8c54-49e4-a40e-8bacdec26f74.png)

**レイテンシー分布**

![](https://user-images.githubusercontent.com/2224492/178294471-b8e1b3c6-a8fd-47cb-872a-0c40418b0da7.png)

**API ゲートウェイ サイドカーの CPU 使用率**

![](https://user-images.githubusercontent.com/2224492/178294732-73aaa9f4-e159-4b8e-ab12-521985313358.png)

**API ゲートウェイ サイドカーのメモリ フットプリント**

![](https://user-images.githubusercontent.com/2224492/178294829-9d2f0794-12e7-4cd6-827d-11af8b632db9.png)

**Bookinfo 評価サイドカーの CPU 使用率**

![](https://user-images.githubusercontent.com/2224492/178295086-6380004f-369d-4f6b-afeb-71b48c0e3053.png)

**Bookinfo 評価サイドカーのメモリ使用量**

![](https://user-images.githubusercontent.com/2224492/178295267-004e7676-04b5-4fef-8e5d-ca196bd7dedc.png)

### サイドカー 1 コア

この違いは、サイドカーを 1 コア CPU に制限するテストで特に顕著です。 この時点で、API Gateway サイドカーがパフォーマンスのボトルネックになり、osm-edge と osm サイドカーの両方で CPU が不足します。

TPS に関しては、osm-edge は 12% 低下し、osm の TPS は驚異的な 65% 低下します。

**TPS**!

![](https://user-images.githubusercontent.com/2224492/178295573-8be92413-d499-476e-b3e1-a23d0bcbcda3.png)

**レイテンシー分布**

![](https://user-images.githubusercontent.com/2224492/178296728-c7ea9a12-d9d4-4be0-9c8d-32bb91724f36.png)

**API ゲートウェイ サイドカーの CPU 使用率**

![](https://user-images.githubusercontent.com/2224492/178300176-0a76080b-3bcb-48f4-a506-4a105ad8c4a8.png)

**API ゲートウェイ サイドカーのメモリ フットプリント**

![](https://user-images.githubusercontent.com/2224492/178300241-95917e41-5857-4a80-8234-ff6533310ef5.png)

**Bookinfo 評価サイドカーの CPU 使用率**

![](https://user-images.githubusercontent.com/2224492/178300596-2f767c75-6872-4aa5-b943-a3eeae84c55e.png)

**Bookinfo 評価サイドカーのメモリ使用量**

![](https://user-images.githubusercontent.com/2224492/178300658-53bdf00d-6f2f-484a-8c3f-0399f9b683ed.png)

## まとめ

今回は、限られたサイドカー リソースで osm-edge と osm データ プレーンのベンチマークを行いました。 結果から、osm-edge は、リソースの使用量が少なく、リソースをより効率的に使用して、依然として高いパフォーマンスを維持できます。 リソースに制約のあるエッジ シナリオでは、クラウドでのみ利用可能なサービス グリッド機能を、より低いリソース オーバーヘッドで利用できます。 これらは、[Pipy](https://flomesh.io) の低リソースで高性能な機能によって可能になります。

もちろん、osm-edge はエッジ コンピューティングのシナリオに適していますが、クラウドにも適用できます。 特に、大規模なサービスを提供するクラウド環境は、コスト管理の要件を満たしています。