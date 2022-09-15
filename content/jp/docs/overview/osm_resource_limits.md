---
Title: 「osm-edge リソースのリクエストと制限」
Description: "osm-edge ポッドおよびデプロイメントのリソース要求と制限"
Type: docs
Weight: 3
---

| 鍵 | タイプ | デフォルト | 説明 |
|-----|------|---------|-------------|
| osm.injector.resource| 物体 | `{"limits":{"cpu":"0.5","memory":"64M"},"requests":{"cpu":"0.3","memory":"64M"}}` | サイドカー インジェクターのコンテナー リソース パラメーター |
| osm.osmBootstrap.resource | 物体 | `{"limits":{"cpu":"0.5","memory":"128M"},"requests":{"cpu":"0.3","memory":"128M"}}` | osm-edge ブートストラップのコンテナ リソース パラメータ |
| osm.osmController.resource | 物体 | `{"limits":{"cpu":"1.5","memory":"1G"},"requests":{"cpu":"0.5","memory":"128M"}}` | osm-edge コントローラのコンテナ リソース パラメータ。 https://docs.openservicemesh.io/docs/guides/ha_scale/scale/ を参照してください。 |
| osm.prometheus.resources | 物体 | `{"limits":{"cpu":"1","memory":"2G"},"requests":{"cpu":"0.5","memory":"512M"}}` | Prometheus のコンテナー リソース パラメーター |

> 注: これらはデフォルト値であり、[values.yaml]((https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/ values.yaml)