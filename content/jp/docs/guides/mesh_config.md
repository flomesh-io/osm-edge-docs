---
title: 「メッシュ構成」
description: 「osm-edge MeshConfig」
type: docs
aliases: ["/docs/osm_mesh_config/"]
weight: 7
---

# osm-edge MeshConfig
osm-edge は MeshConfig リソース `osm-mesh-config` をそのコントロール プレーン (osm-controller pod と同じ名前空間内) の一部としてデプロイします。これは、メッシュの所有者/オペレーターがいつでも更新できます。 この MeshConfig の目的は、メッシュの所有者/オペレーターが、ニーズに基づいてメッシュ構成の一部を更新できるようにすることです。

インストール時に、osm-edge MeshConfig は、[charts/osm/templates](https://github.com/flomesh-io) の下にあるプリセット MeshConfig (`preset-mesh-config`) から展開されます。 /osm-edge/blob/{{< param osm_branch >}}/charts/osm/templates/preset-mesh-config.yaml)。

まず、osm がインストールされた名前空間を参照するように環境変数を設定します。
```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where osm-edge is installed
```

CLI で「osm-mesh-config」を表示するには、「kubectl get」コマンドを使用します。

```bash
kubectl get meshconfig osm-mesh-config -n "$osm_namespace" -o yaml
```

*注: MeshConfig `osm-mesh-config` の値は、アップグレード後も保持されます。*

## osm-edge MeshConfig を構成する

### Kubectl パッチ コマンド

「osm-mesh-config」への変更は、「kubectl patch」コマンドを使用して行うことができます。
```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge
```
詳細については、[Config API reference](/docs/apidocs/config/v1alpha1) を参照してください。

誤った値が使用された場合、[MeshConfig CRD](https://github.com/flomesh-io/osm-edge/blob/{{< param osm_branch >}}/charts/osm/crds/meshconfig. yaml) は、値が無効である理由を説明するエラー メッセージで変更を防ぎます。
たとえば、以下のコマンドは、「enableEgress」をブール値以外の値にパッチするとどうなるかを示しています。
```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":"no"}}}'  --type=merge
# Validations on the CRD will deny this change
The MeshConfig "osm-mesh-config" is invalid: spec.traffic.enableEgress: Invalid value: "string": spec.traffic.enableEgress in body must be of type boolean: "string"
```
#### キーの種類ごとの kubectl パッチ コマンド

> 注: `<osm-namespace>` は、osm コントロール プレーンがインストールされている名前空間を指します。 デフォルトでは、osm 名前空間は「osm-system」です.

| キー                                            | タイプ   | デフォルト値                                | kubectl パッチ コマンドの例                                                                                                     |
| ---------------------------------------------- | ------ | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| spec.traffic.enableEgress                      | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge`                                                     |
| spec.traffic.enablePermissiveTrafficPolicyMode | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge`                                |
| spec.traffic.useHTTPSIngress                   | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"useHTTPSIngress":true}}}'  --type=merge`                                                  |
| spec.traffic.outboundPortExclusionList         | array  | `[]`                                         | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"outboundPortExclusionList":6379,8080}}}'  --type=merge`                                   |
| spec.traffic.outboundIPRangeExclusionList      | array  | `[]`                                         | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":"10.0.0.0/32,1.1.1.1/24"}}}'  --type=merge`                 |
| spec.certificate.serviceCertValidityDuration   | string | `"24h"`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"certificate":{"serviceCertValidityDuration":"24h"}}}'  --type=merge`                                 |
| spec.observability.enableDebugServer           | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"serviceCertValidityDuration":true}}}'  --type=merge`                                |
| spec.observability.tracing.enable              | bool   | `"jaeger.<osm-namespace>.svc.cluster.local"` | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"tracing":{"address": "jaeger.<osm-namespace>.svc.cluster.local"}}}}'  --type=merge` |
| spec.observability.tracing.address             | string | `"/api/v2/spans"`                            | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"tracing":{"endpoint":"/api/v2/spans"}}}}'  --type=merge' --type=merge`              |
| spec.observability.tracing.endpoint            | string | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"tracing":{"enable":true}}}}'  --type=merge`                                         |
| spec.observability.tracing.port                | int    | `9411`                                       | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"tracing":{"port":9411}}}}'  --type=merge`                                           |
| spec.sidecar.enablePrivilegedInitContainer     | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"enablePrivilegedInitContainer":true}}}'  --type=merge`                                    |
| spec.sidecar.logLevel                          | string | `"error"`                                    | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"logLevel":"error"}}}'  --type=merge`                                                      |
| spec.sidecar.maxDataPlaneConnections           | int    | `0`                                          | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"maxDataPlaneConnections":"error"}}}'  --type=merge`                                       |
| spec.sidecar.configResyncInterval              | string | `"0s"`                                       | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"configResyncInterval":"30s"}}}'  --type=merge`                                            |
