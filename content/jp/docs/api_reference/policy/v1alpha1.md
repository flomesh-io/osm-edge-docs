---
title: "ポリシー v1alpha1 API リファレンス"
description: "ポリシー v1alpha1 API リファレンス ドキュメント。"
type: docs
---

<p>パッケージ:</p>
<ul>
<li>
<a href="#policy.openservicemesh.io%2fv1alpha1">policy.openservicemesh.io/v1alpha1</a>
</li>
</ul>
<h2 id="policy.openservicemesh.io/v1alpha1">policy.openservicemesh.io/v1alpha1</h2>
<div>
<p>パッケージv1alpha1 は、APIのv1alpha1バージョンだ。</p>
</div>
リソースタイプ:
<ul></ul>
<h3 id="policy.openservicemesh.io/v1alpha1.BackendSpec">バックエンド仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackendSpec">IngressBackendSpecに表示される</a>)
</p>
<div>
<p>バックエンド仕様は、イングレスバックエンドポリシー仕様で指定されたバックエンドを表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>名称</code><br/>
<em>
ストリング
</em>
</td>
<td>
<p>名称は、バックエンドの名前を指す。</p>
</td>
</tr>
<tr>
<td>
<code>ポート</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.PortSpec">
ポート仕様
</a>
</em>
</td>
<td>
<p>ポートは、バックエンドのポートの仕様を指す。</p>
</td>
</tr>
<tr>
<td>
<code>tls</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.TLSSpec">
TLS仕様
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>TLS は、バックエンドの TLS 設定の仕様を指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">接続設定仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec">UpstreamTrafficSettingSpecに表示される</a>)
</p>
<div>
<p>ConnectionSettingsSpecは、アップストリームホストの接続設定を指す。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>tcp</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.TCPConnectionSettings">
TCP接続設定
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>TCP は、TCP レベルの接続設定を指定する。TCP 接続と HTTP 接続の両方に適用される。</p>
</td>
</tr>
<tr>
<td>
<code>http</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.HTTPConnectionSettings">
HTTP接続設定
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>HTTP は、HTTP レベルの接続設定を指定する。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.Egress">エグレス
</h3>
<div>
<p>Egress は、Egressトラフィックポリシーを表すために使用されるタイプだ。Egress ポリシーは、ポリシーで指定されたルールに基づいて、アプリケーションがサービスメッシュまたはクラスターの外部のエンドポイントにアクセスできるようにする。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>メタデータ</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#objectmeta-v1-meta">
Kubernetes meta/v1.ObjectMeta
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>オブジェクトのメタデータ</p>
<code>metadata</code> フィールドのフィールドについては、Kubernetes API ドキュメントを参照してください。
</td>
</tr>
<tr>
<td>
<code>仕様</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.EgressSpec">
EgressSpec
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Specは、エグレス ポリシーの仕様だ。</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>ソース</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.EgressSourceSpec">
[]EgressSourceSpec
</a>
</em>
</td>
<td>
<p>Sources は、エグレスポリシーが適用されるソースのリストを指す。</p>
</td>
</tr>
<tr>
<td>
<code>ホスト</code><br/>
<em>
[]ストリング
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Hostsは、エグレスポリシーがアクセスを許可する外部ホストのリストを指す。</p>
<ul>
<li><p>HTTPトラフィックの場合、HTTP Host/Authority ヘッダーは、指定されたホストのリストと照合される。</p></li>
<li><p>HTTPS トラフィックの場合、TLS ハンドシェークでクライアントによって示された Server Name Indication (SNI) が、指定されたホストのリストと照合される。</p></li>
<li><p>非 HTTP(s) ベースのプロトコルの場合、Hostsフィールドは無視される。</p></li>
</ul>
</td>
</tr>
<tr>
<td>
<code>ipアドレス</code><br/>
<em>
[]ストリング
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>IPアドレスは、エグレスポリシーが適用される外部 IPアドレス範囲のリストを指す。トラフィックの宛先 IP アドレスは、CIDR 範囲として指定された IP アドレスのリストと照合される。</p>
</td>
</tr>
<tr>
<td>
<code>ポート</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.PortSpec">
[]PortSpec
</a>
</em>
</td>
<td>
<p>ポートは、エグレスポリシーが適用されるポートのリストを指す。トラフィックの宛先ポートは、指定されたポートのリストと照合される。</p>
</td>
</tr>
<tr>
<td>
<code>マッチ</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#typedlocalobjectreference-v1-core">
[]Kubernetes core/v1.TypedLocalObjectReference
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Matches は、送信ポリシーが一致する必要があるオブジェクト参照のリストを指す。</p>
</td>
</tr>
</table>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.EgressSourceSpec">エグレスソース仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.EgressSpec">EgressSpecに表示される</a>)
</p>
<div>
<p>EgressSourceSpec は、エグレスポリシー仕様で指定されたソースのリストでソースを表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>種類</code><br/>
<em>
ストリング
</em>
</td>
<td>
<p>Kindは、エグレスポリシーでソースの種類を指す。例えば、ServiceAccount。</p>
</td>
</tr>
<tr>
<td>
<code>名称</code><br/>
<em>
ストリング
</em>
</td>
<td>
<p>Nameは、指定された Kind のソースの名前を指す。</p>
</td>
</tr>
<tr>
<td>
<code>名前空間</code><br/>
<em>
ストリング
</em>
</td>
<td>
<p>Namespaceは、指定されたソースの名前空間を指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.EgressSpec">エグレス仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.Egress">Egressに表示される</a>)
</p>
<div>
<p>EgressSpecは、エグレスポリシー仕様を表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>ソース</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.EgressSourceSpec">
[]EgressSourceSpec
</a>
</em>
</td>
<td>
<p>Sourcesは、エグレスポリシーが適用されるソースのリストを指す。</p>
</td>
</tr>
<tr>
<td>
<code>ホスト</code><br/>
<em>
[]string
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>ホストは、エグレスポリシーがアクセスを許可する外部ホストのリストを指す。</p>
<ul>
<li><p>HTTPトラフィックの場合、HTTP Host/Authority ヘッダーは、指定されたホストのリストと照合される。</p></li>
<li><p>HTTPS トラフィックの場合、TLS ハンドシェークでクライアントによって示された Server Name Indication (SNI) が、指定されたホストのリストと照合される。</p></li>
<li><p>非 HTTP(s) ベースのプロトコルの場合、Hostsフィールドは無視される。</p></li>
</ul>
</td>
</tr>
<tr>
<td>
<code>ipアドレス</code><br/>
<em>
[]string
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>IPAddresses は、エグレスポリシーが適用される外部 IP アドレス範囲のリストを指す。トラフィックの宛先 IP アドレスは、CIDR 範囲として指定されたIPアドレスのリストと照合される。</p>
</td>
</tr>
<tr>
<td>
<code>ポート</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.PortSpec">
[]PortSpec
</a>
</em>
</td>
<td>
<p>Ports は、エグレスポリシーが適用されるポートのリストを指す。トラフィックの宛先ポートは、指定されたポートのリストと照合される。</p>
</td>
</tr>
<tr>
<td>
<code>マッチ</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#typedlocalobjectreference-v1-core">
[]Kubernetes core/v1.TypedLocalObjectReference
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Matches は、送信ポリシーが一致する必要があるオブジェクト参照のリストを指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.HTTPConnectionSettings">HTTP接続設定
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">ConnectionSettingsSpecに表示される</a>)
</p>
<div>
<p>HTTPConnectionSettings は、アップストリームホストの HTTP 接続設定を指す。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>maxRequests</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>MaxRequestsは、アップストリームホストに許可される並列リクエストの最大数を指す。指定しない場合、デフォルトは 4294967295 (2^32 - 1) だ。</p>
</td>
</tr>
<tr>
<td>
<code>maxRequestsPerConnection</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>MaxRequestsPerConnection は、アップストリームホストに許可される接続ごとのリクエストの最大数を指す。指定しない場合、デフォルトで無制限になる。</p>
</td>
</tr>
<tr>
<td>
<code>maxPendingRequests</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>MaxPendingRequests は、アップストリーム ホストに許可される保留中の HTTP リクエストの最大数を指す。HTTP/2 接続の場合、maxRequestsPerConnection が設定されていない場合、すべてのリクエストが同じ接続を介して多重化されるため、このサーキットブレーカーは、接続がまだ確立されていない場合にのみヒットする。指定しない場合、デフォルトは 4294967295 (2^32 - 1) だ。</p>
</td>
</tr>
<tr>
<td>
<code>maxRetries</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>MaxRetries は、アップストリームホストに許可される並列再試行の最大数を指す。指定しない場合、デフォルトは 4294967295 (2^32 - 1) だ。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.IngressBackend">イングレスバックエンド
</h3>
<div>
<p>IngressBackend は、イングレスバックエンド ポリシーを表すために使用されるタイプだ。イングレスバックエンドポリシーは、1 つ以上のソースからのイングレストラフィックを受け入れるように1 つ以上のバックエンドを認証する。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>metadata</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#objectmeta-v1-meta">
Kubernetes meta/v1.ObjectMeta
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>オブジェクトのメタデータ</p>
<code>metadata</code>フィールドのフィールドについては、Kubernetes API ドキュメントを参照してください。
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.IngressBackendSpec">
IngressBackendSpec
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Spec はイングレスバックエンドポリシーの仕様だ</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>backends</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">
[]BackendSpec
</a>
</em>
</td>
<td>
<p>Backends は、IngressBackendポリシーが適用されるバックエンドのリストを指す。</p>
</td>
</tr>
<tr>
<td>
<code>sources</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.IngressSourceSpec">
[]IngressSourceSpec
</a>
</em>
</td>
<td>
<p>Sources は、IngressBackendポリシーが適用されるソースのリストを指す。</p>
</td>
</tr>
<tr>
<td>
<code>matches</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#typedlocalobjectreference-v1-core">
[]Kubernetes core/v1.TypedLocalObjectReference
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Matches は、IngressBackend ポリシーが一致するオブジェクト参照のリストを指す。</p>
</td>
</tr>
</table>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.IngressBackendStatus">
IngressBackendStatus
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Status は、IngressBackend構成のステータスだ。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.IngressBackendSpec">イングレスバックエンド仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackend">IngressBackendに表示される</a>)
</p>
<div>
<p>IngressBackendSpec は、IngressBackendポリシー仕様を表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>backends</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">
[]BackendSpec
</a>
</em>
</td>
<td>
<p>Backends は、IngressBackendポリシーが適用されるバックエンドのリストを指す。</p>
</td>
</tr>
<tr>
<td>
<code>sources</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.IngressSourceSpec">
[]IngressSourceSpec
</a>
</em>
</td>
<td>
<p>Sources は、IngressBackend ポリシーが適用されるソースのリストを指す。</p>
</td>
</tr>
<tr>
<td>
<code>matches</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#typedlocalobjectreference-v1-core">
[]Kubernetes core/v1.TypedLocalObjectReference
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Matches は、IngressBackend ポリシーが一致するオブジェクト参照のリストを指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.IngressBackendStatus">イングレスバックエンドステータス
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackend">IngressBackendに表示される</a>)
</p>
<div>
<p>IngressBackendStatus は、IngressBackend リソースのステータスを表すために使用されるタイプだ。p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>currentStatus</code><br/>
<em>
string
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>CurrentStatus は、IngressBackendリソースの現在のステータスを指す。</p>
</td>
</tr>
<tr>
<td>
<code>reason</code><br/>
<em>
string
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Reason は、IngressBackend リソースの現在のステータスの理由を指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.IngressSourceSpec">イングレスソース仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackendSpec">IngressBackendSpecに表示される</a>)
</p>
<div>
<p>IngressSourceSpec は、IngressBackend ポリシー仕様で指定されたソースのリストでソースを表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>kind</code><br/>
<em>
string
</em>
</td>
<td>
<p>Kind は、IngressBackend ポリシーでソースの種類を指す。次のいずれかである必要がある: Service、AuthenticatedPrincipal、IPRange</p>
</td>
</tr>
<tr>
<td>
<code>name</code><br/>
<em>
string
</em>
</td>
<td>
<p>Name は、指定された Kind のソースの名前を指す。</p>
</td>
</tr>
<tr>
<td>
<code>namespace</code><br/>
<em>
string
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Namespace は、指定されたソースの名前空間を指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.PortSpec">ポート仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">BackendSpec</a>, <a href="#policy.openservicemesh.io/v1alpha1.EgressSpec">EgressSpecに表示される</a>)
</p>
<div>
<p>PortSpec は、Egress ポリシー仕様で指定されたポートのリスト内のポートを表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>number</code><br/>
<em>
int
</em>
</td>
<td>
<p>Number は、ポート番号を指す。</p>
</td>
</tr>
<tr>
<td>
<code>protocol</code><br/>
<em>
string
</em>
</td>
<td>
<p>Protocol は、ポートが提供するプロトコルを指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.Retry">再試行
</h3>
<div>
<p>Retry は、再試行ポリシーを表すために使用されるタイプだ。再試行ポリシーは、1 つのサービスソースから 1 つ以上の宛先サービスへのアウトバウンドトラフィックの失敗した試行に対する再試行を許可する。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>metadata</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#objectmeta-v1-meta">
Kubernetes meta/v1.ObjectMeta
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>オブジェクトのメタデータ</p>
<code>metadata</code> フィールドのフィールドについては、Kubernetes API ドキュメントを参照してください。
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySpec">
RetrySpec
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Spec は再試行ポリシーの仕様だ</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>source</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">
RetrySrcDstSpec
</a>
</em>
</td>
<td>
<p>sourceは、再試行ポリシーが適用されるソースを指す。</p>
</td>
</tr>
<tr>
<td>
<code>destinations</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">
[]RetrySrcDstSpec
</a>
</em>
</td>
<td>
<p>Destinations は、再試行ポリシーが適用される宛先のリストを指す。</p>
</td>
</tr>
<tr>
<td>
<code>retryPolicy</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetryPolicySpec">
RetryPolicySpec
</a>
</em>
</td>
<td>
<p>RetryPolicy は、再試行ポリシーが適用される再試行ポリシーを指す。</p>
</td>
</tr>
</table>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.RetryPolicySpec">再試行ポリシー仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.RetrySpec">RetrySpecに表示される</a>)
</p>
<div>
<p>RetryPolicySpec は、再試行ポリシー仕様で指定された再試行ポリシーを表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>retryOn</code><br/>
<em>
string
</em>
</td>
<td>
<p>RetryOn は、再試行するポリシーをコンマで区切って指す。</p>
</td>
</tr>
<tr>
<td>
<code>perTryTimeout</code><br/>
<em>
string
</em>
</td>
<td>
<p>PerTryTimeout は、試行の失敗と見なされるまでの再試行に許可される時間を指す。</p>
</td>
</tr>
<tr>
<td>
<code>numRetries</code><br/>
<em>
int
</em>
</td>
<td>
<p>NumRetries は、試行する再試行の最大回数を指す。</p>
</td>
</tr>
<tr>
<td>
<code>retryBackoffInterval</code><br/>
<em>
string
</em>
</td>
<td>
<p>RetryBackoffBase Interval は、指数再試行バックオフの基本間隔を指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.RetrySpec">再試行仕様
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.Retry">Retryに表示される</a>)
</p>
<div>
<p>RetrySpec は、再試行ポリシーの仕様を表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>source</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">
RetrySrcDstSpec
</a>
</em>
</td>
<td>
<p>Sourceは、再試行ポリシーが適用されるソースを指す。</p>
</td>
</tr>
<tr>
<td>
<code>destinations</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">
[]RetrySrcDstSpec
</a>
</em>
</td>
<td>
<p>Destinations は、再試行ポリシーが適用される宛先のリストを指す。</p>
</td>
</tr>
<tr>
<td>
<code>retryPolicy</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetryPolicySpec">
RetryPolicySpec
</a>
</em>
</td>
<td>
<p>RetryPolicy は、再試行ポリシーが適用される再試行ポリシーを指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">RetrySrcDstSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.RetrySpec">RetrySpecに表示される</a>)
</p>
<div>
<p>RetrySrcDstSpec は、Destinationリスト内の Destinationと、再試行ポリシー仕様で指定された Source を表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code></code><br/>
<em>
string
</em>
</td>
<td>
<p>Kind は、Retryポリシーの Src/Dst の種類を指す。</p>
</td>
</tr>
<tr>
<td>
<code>name</code><br/>
<em>
string
</em>
</td>
<td>
<p>Name は、指定された Kind の Src/Dst の名前を指す。</p>
</td>
</tr>
<tr>
<td>
<code>namespace</code><br/>
<em>
string
</em>
</td>
<td>
<p>Namespace は、指定された Src/Dst の名前空間を指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.TCPConnectionSettings">TCP接続設定
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">ConnectionSettingsSpecに表示される</a>)
</p>
<div>
<p>TCPConnectionSettings は、アップストリームホストの TCP 接続設定を指す。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>maxConnections</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>MaxConnections は、アップストリームホストに許可される TCP 接続の最大数を指す。指定しない場合、デフォルトは 4294967295 (2^32 - 1) だ。
</p>
</td>
</tr>
<tr>
<td>
<code>connectTimeout</code><br/>
<em>
<a href="https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#Duration">
Kubernetes meta/v1.Duration
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>ConnectTimeout は、TCP 接続タイムアウトを指す。指定しない場合、デフォルトは 5 秒だ。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.TLSSpec">TLS仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">BackendSpecに表示される</a>)
</p>
<div>
<p>TLSSpec は、バックエンドの TLS 構成を表すために使用されるタイプだ。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>skipClientCertValidation</code><br/>
<em>
bool
</em>
</td>
<td>
<p>SkipClientCertValidation は、バックエンドがクライアントによって提示された証明書の検証をスキップする必要があるかどうかを決める。</p>
</td>
</tr>
<tr>
<td>
<code>sniHosts</code><br/>
<em>
[]string
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>SNIHosts は、バックエンドがクライアントの接続を許可する SNIホスト名を指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.UpstreamTrafficSetting">アップストリームトラフィック設定
</h3>
<div>
<p>UpstreamTrafficSetting は、アップストリームホスト宛てのトラフィックに適用可能な設定を指す。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>metadata</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#objectmeta-v1-meta">
Kubernetes meta/v1.ObjectMeta
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>オブジェクトのメタデータ</p>
<code>metadata</code>フィールドのフィールドについては、Kubernetes API ドキュメントを参照してください。
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec">
UpstreamTrafficSettingSpec
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Spec は UpstreamTrafficSetting ポリシーの仕様だ</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>host</code><br/>
<em>
string
</em>
</td>
<td>
<p>アップストリーム トラフィックが送信されるホスト。
アップストリーム サービスに対応する FQDN またはアップストリーム サービスの名前のいずれかである必要がある。サービス名のみを指定した場合は、サービス名とUpstreamTrafficSettingルールの名前空間からFQDNを導出する。</p>
</td>
</tr>
<tr>
<td>
<code>connectionSettings</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">
ConnectionSettingsSpec
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>ConnectionSettings は、アップストリームホストに送信されるトラフィックの接続設定を指す。</p>
</td>
</tr>
</table>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingStatus">
UpstreamTrafficSettingStatus
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Status は UpstreamTrafficSetting リソースのステータスだ。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec">アップストリームトラフィック設定仕様
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSetting">UpstreamTrafficSettingに表示される</a>)
</p>
<div>
<p>UpstreamTrafficSettingSpec は、アップストリームトラフィックの設定仕様を指す。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>host</code><br/>
<em>
string
</em>
</td>
<td>
<p>アップストリーム トラフィックが送信されるホスト。アップストリームサービスに対応する FQDN またはアップストリーム サービスの名前のいずれかである必要がある。サービス名のみを指定した場合は、サービス名とUpstreamTrafficSettingルールの名前空間からFQDNを導出する。</p>
</td>
</tr>
<tr>
<td>
<code>connectionSettings</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">
ConnectionSettingsSpec
</a>
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>ConnectionSettings は、アップストリームホストに送信されるトラフィックの接続設定を指す。</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingStatus">アップストリームトラフィック設定ステータス
</h3>
<p>
(<em></em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSetting">UpstreamTrafficSettingに表示される</a>)
</p>
<div>
<p>UpstreamTrafficSettingStatus は、UpstreamTrafficSetting リソースのステータスを指す。</p>
</div>
<table>
<thead>
<tr>
<th>フィールド</th>
<th>説明</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>currentStatus</code><br/>
<em>
string
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>CurrentStatus は、UpstreamTrafficSetting リソースの現在のステータスを指す。</p>
</td>
</tr>
<tr>
<td>
<code>reason</code><br/>
<em>
string
</em>
</td>
<td>
<em>（オプショナル）</em>
<p>Reason は、UpstreamTrafficSetting リソースの現在のステータスの理由を指す。</p>
</td>
</tr>
</tbody>
</table>
<hr/>
<p><em>
git commit <code>407bbedd5</code> の <code>gen-crd-api-reference-docs</code> で生成された。
</em></p>
