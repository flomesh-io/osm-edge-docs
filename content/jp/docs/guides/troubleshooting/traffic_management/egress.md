---
title: 「出口のトラブルシューティング」
description: 「エグレストラブルシューティングガイド」
type: docs
---

## Egress が期待どおりに機能しない場合

### 1. エグレスが有効になっていることを確認する

「osm-mesh-config」「MeshConfig」カスタム リソースの「enableEgress」キーの値を確認して、エグレスが有効になっていることを確認します。 `osm-mesh-config` は名前空間 osm-edge コントロール プレーンの名前空間 (デフォルトでは `osm-system`) にあります。

```console
# Returns true if egress is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enableEgress}{"\n"}'
true
```

上記のコマンドは、エグレスが有効かどうかを示すブール文字列 (`true` または `false`) を返す必要があります。

### 2. osm-edge コントローラーのログでエラーを調べます

```bash
# When osm-controller is deployed in the osm-system namespace
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

エラーは、ログ メッセージの「レベル」キーを「エラー」に設定してログに記録されます。
```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. Pipy の構成を確認する

Pod のサイドカーで使用される構成で egress が有効になっていることを確認します。

```json
{
  "Spec": {
    "SidecarLogLevel": "error",
    "Traffic": {
      "EnableEgress": true
    }
  }
}
```
