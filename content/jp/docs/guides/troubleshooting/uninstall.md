---
title: 「アンインストールのトラブルシューティング」
description: 「osm-edge メッシュ アンインストール トラブルシューティング ガイド」
aliases: ["/docs/troubleshooting/uninstall"]
type: docs
---

# osm-edge メッシュ アンインストール トラブルシューティング ガイド

何らかの理由で「osm uninstall mesh」([アンインストール ガイド](/docs/guides/uninstall/) に記載されている) が失敗した場合は、以下で説明するように osm-edge リソースを手動で削除できます。

メッシュの環境変数を設定します。
```console
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
export mesh_name=osm # Replace osm with the osm-edge mesh name
export osm_version=<osm version>
export osm_ca_bundle=<osm ca bundle>
```

Delete osm-edge control plane deployments:
```console
kubectl delete deployment -n $osm_namespace osm-bootstrap
kubectl delete deployment -n $osm_namespace osm-controller
kubectl delete deployment -n $osm_namespace osm-injector
```

Prometheus、Grafana、または Jaeger と一緒に osm-edge がインストールされている場合は、それらのデプロイメントを削除します。
```console
kubectl delete deployment -n $osm_namespace osm-prometheus
kubectl delete deployment -n $osm_namespace osm-grafana
kubectl delete deployment -n $osm_namespace jaeger
```

osm-edge が osm-edge マルチクラスター ゲートウェイと共にインストールされている場合は、次のコマンドを実行して削除します。
```console
kubectl delete deployment -n $osm_namespace osm-multicluster-gateway
```

osm-edge シークレット、meshconfig、および webhook 構成を削除します。
> 警告: 続行する前に、クラスタ内のリソースが次のリソースに依存していないことを確認してください。
```console
kubectl delete secret -n $osm_namespace $osm_ca_bundle mutating-webhook-cert-secret validating-webhook-cert-secret crd-converter-cert-secret
kubectl delete meshconfig -n $osm_namespace osm-mesh-config
kubectl delete mutatingwebhookconfiguration -l app.kubernetes.io/name=openservicemesh.io,app.kubernetes.io/instance=$mesh_name,app.kubernetes.io/version=$osm_version,app=osm-injector
kubectl delete validatingwebhookconfiguration -l app.kubernetes.io/name=openservicemesh.io,app.kubernetes.io/instance=mesh_name,app.kubernetes.io/version=$osm_version,app=osm-controller
```

クラスターから osm-edge および SMI CRD を削除するには、次を実行します。
> 警告: CRD を削除すると、その CRD に対応するすべてのカスタム リソースも削除されます。
```console
kubectl delete crd meshconfigs.config.openservicemesh.io
kubectl delete crd multiclusterservices.config.openservicemesh.io
kubectl delete crd egresses.policy.openservicemesh.io
kubectl delete crd ingressbackends.policy.openservicemesh.io
kubectl delete crd httproutegroups.specs.smi-spec.io
kubectl delete crd tcproutes.specs.smi-spec.io
kubectl delete crd traffictargets.access.smi-spec.io
kubectl delete crd trafficsplits.split.smi-spec.io
```