---
title: 「許容トラフィック ポリシー モードのトラブルシューティング」
description: 「許可トラフィック ポリシー モードのトラブルシューティング ガイド」
type: docs
---

## permissive トラフィック ポリシー モードが期待どおりに機能しない場合

### 1. permissive トラフィック ポリシー モードが有効になっていることを確認する

「osm-mesh-config」カスタム リソースの「enablePermissiveTrafficPolicyMode」キーの値を確認して、寛容なトラフィック ポリシー モードが有効になっていることを確認します。 `osm-mesh-config` MeshConfig は名前空間 osm-edge コントロール プレーンの名前空間 (デフォルトでは `osm-system`) にあります。

```console
# Returns true if permissive traffic policy mode is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enablePermissiveTrafficPolicyMode}{"\n"}'
true
```

上記のコマンドは、許容トラフィック ポリシー モードが有効かどうかを示すブール文字列 (「true」または「false」) を返す必要があります。

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

クライアントの Pipy プロキシ構成とサーバー Pod で、クライアントがサーバーにアクセスできることを確認します。 