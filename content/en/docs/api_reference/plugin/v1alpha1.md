---
title: "Plugin v1alpha1 API Reference"
description: "Plugin v1alpha1 API reference documentation."
type: docs
---

<p>Packages:</p>
<ul>
<li>
<a href="#plugin.flomesh.io%2fv1alpha1">plugin.flomesh.io/v1alpha1</a>
</li>
</ul>
<h2 id="plugin.flomesh.io/v1alpha1">plugin.flomesh.io/v1alpha1</h2>
<div>
<p>Package v1alpha1 is the v1alpha1 version of the API.</p>
</div>
Resource Types:
<ul></ul>
<h3 id="plugin.flomesh.io/v1alpha1.ChainPluginSpec">ChainPluginSpec
</h3>
<p>
(<em>Appears on:</em><a href="#plugin.flomesh.io/v1alpha1.PluginChainSpec">PluginChainSpec</a>)
</p>
<div>
<p>ChainPluginSpec is the type used to represent plugins within chain.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>name</code><br/>
<em>
string
</em>
</td>
<td>
<p>Name defines the name of chain.</p>
</td>
</tr>
<tr>
<td>
<code>plugins</code><br/>
<em>
[]string
</em>
</td>
<td>
<p>Plugins defines the plugins within chain.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.ChainSelectorSpec">ChainSelectorSpec
</h3>
<p>
(<em>Appears on:</em><a href="#plugin.flomesh.io/v1alpha1.PluginChainSpec">PluginChainSpec</a>)
</p>
<div>
<p>ChainSelectorSpec is the type used to represent plugins for plugin chain.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>podSelector</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#labelselector-v1-meta">
Kubernetes meta/v1.LabelSelector
</a>
</em>
</td>
<td>
<p>PodSelector for pods. Existing pods are selected by this will be the ones affected by this plugin chain.</p>
</td>
</tr>
<tr>
<td>
<code>namespaceSelector</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#labelselector-v1-meta">
Kubernetes meta/v1.LabelSelector
</a>
</em>
</td>
<td>
<p>NamespaceSelector for namespaces. Existing pods are selected by this will be the ones affected by this plugin chain.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.Plugin">Plugin
</h3>
<div>
<p>Plugin is the type used to represent a Plugin policy.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
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
<em>(Optional)</em>
<p>Object&rsquo;s metadata</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.PluginSpec">
PluginSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the PlugIn specification</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>priority</code><br/>
<em>
float32
</em>
</td>
<td>
<p>priority defines the priority of the plugin.</p>
</td>
</tr>
<tr>
<td>
<code>pipyscript</code><br/>
<em>
string
</em>
</td>
<td>
<p>Script defines the Script of the plugin.</p>
</td>
</tr>
</table>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.PluginStatus">
PluginStatus
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Status is the status of the Plugin configuration.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.PluginChain">PluginChain
</h3>
<div>
<p>PluginChain is the type used to represent a PluginChain.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
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
<em>(Optional)</em>
<p>Object&rsquo;s metadata</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.PluginChainSpec">
PluginChainSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the PluginChain specification</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>chains</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.ChainPluginSpec">
[]ChainPluginSpec
</a>
</em>
</td>
<td>
<p>Chains defines the plugins within chains</p>
</td>
</tr>
<tr>
<td>
<code>selectors</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.ChainSelectorSpec">
ChainSelectorSpec
</a>
</em>
</td>
<td>
<p>Selectors defines the selectors of chains.</p>
</td>
</tr>
</table>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.PluginChainStatus">
PluginChainStatus
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Status is the status of the PluginChain configuration.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.PluginChainSpec">PluginChainSpec
</h3>
<p>
(<em>Appears on:</em><a href="#plugin.flomesh.io/v1alpha1.PluginChain">PluginChain</a>)
</p>
<div>
<p>PluginChainSpec is the type used to represent the PluginChain specification.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>chains</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.ChainPluginSpec">
[]ChainPluginSpec
</a>
</em>
</td>
<td>
<p>Chains defines the plugins within chains</p>
</td>
</tr>
<tr>
<td>
<code>selectors</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.ChainSelectorSpec">
ChainSelectorSpec
</a>
</em>
</td>
<td>
<p>Selectors defines the selectors of chains.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.PluginChainStatus">PluginChainStatus
</h3>
<p>
(<em>Appears on:</em><a href="#plugin.flomesh.io/v1alpha1.PluginChain">PluginChain</a>)
</p>
<div>
<p>PluginChainStatus is the type used to represent the status of a PluginChain resource.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
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
<em>(Optional)</em>
<p>CurrentStatus defines the current status of an PluginChain resource.</p>
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
<em>(Optional)</em>
<p>Reason defines the reason for the current status of an PluginChain resource.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.PluginConfig">PluginConfig
</h3>
<div>
<p>PluginConfig is the type used to represent a plugin config policy.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
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
<em>(Optional)</em>
<p>Object&rsquo;s metadata</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.PluginConfigSpec">
PluginConfigSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the PlugIn specification</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>plugin</code><br/>
<em>
string
</em>
</td>
<td>
<p>Plugin is the name of plugin.</p>
</td>
</tr>
<tr>
<td>
<code>destinationRefs</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#objectreference-v1-core">
[]Kubernetes core/v1.ObjectReference
</a>
</em>
</td>
<td>
<p>DestinationRefs is the destination references of plugin.</p>
</td>
</tr>
<tr>
<td>
<code>config</code><br/>
<em>
k8s.io/apimachinery/pkg/runtime.RawExtension
</em>
</td>
<td>
<p>Config is the config of plugin.</p>
</td>
</tr>
</table>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#plugin.flomesh.io/v1alpha1.PluginConfigStatus">
PluginConfigStatus
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Status is the status of the plugin config configuration.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.PluginConfigSpec">PluginConfigSpec
</h3>
<p>
(<em>Appears on:</em><a href="#plugin.flomesh.io/v1alpha1.PluginConfig">PluginConfig</a>)
</p>
<div>
<p>PluginConfigSpec is the type used to represent the plugin config specification.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>plugin</code><br/>
<em>
string
</em>
</td>
<td>
<p>Plugin is the name of plugin.</p>
</td>
</tr>
<tr>
<td>
<code>destinationRefs</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#objectreference-v1-core">
[]Kubernetes core/v1.ObjectReference
</a>
</em>
</td>
<td>
<p>DestinationRefs is the destination references of plugin.</p>
</td>
</tr>
<tr>
<td>
<code>config</code><br/>
<em>
k8s.io/apimachinery/pkg/runtime.RawExtension
</em>
</td>
<td>
<p>Config is the config of plugin.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.PluginConfigStatus">PluginConfigStatus
</h3>
<p>
(<em>Appears on:</em><a href="#plugin.flomesh.io/v1alpha1.PluginConfig">PluginConfig</a>)
</p>
<div>
<p>PluginConfigStatus is the type used to represent the status of a PluginConfig resource.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
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
<em>(Optional)</em>
<p>CurrentStatus defines the current status of an PluginConfig resource.</p>
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
<em>(Optional)</em>
<p>Reason defines the reason for the current status of an PluginConfig resource.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.PluginSpec">PluginSpec
</h3>
<p>
(<em>Appears on:</em><a href="#plugin.flomesh.io/v1alpha1.Plugin">Plugin</a>)
</p>
<div>
<p>PluginSpec is the type used to represent the Plugin policy specification.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>priority</code><br/>
<em>
float32
</em>
</td>
<td>
<p>priority defines the priority of the plugin.</p>
</td>
</tr>
<tr>
<td>
<code>pipyscript</code><br/>
<em>
string
</em>
</td>
<td>
<p>Script defines the Script of the plugin.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="plugin.flomesh.io/v1alpha1.PluginStatus">PluginStatus
</h3>
<p>
(<em>Appears on:</em><a href="#plugin.flomesh.io/v1alpha1.Plugin">Plugin</a>)
</p>
<div>
<p>PluginStatus is the type used to represent the status of a Plugin resource.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
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
<em>(Optional)</em>
<p>CurrentStatus defines the current status of an Plugin resource.</p>
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
<em>(Optional)</em>
<p>Reason defines the reason for the current status of an Plugin resource.</p>
</td>
</tr>
</tbody>
</table>
<hr/>
<p><em>
Generated with <code>gen-crd-api-reference-docs</code>
on git commit <code>06a57e32</code>.
</em></p>
