---
title: "Extending osm-edge with Plugins"
description: "Adding Identity and Access Management functionality to osm-edge sidecar via Plugins"
type: docs
weight: 25
---

In this demonstration, we will extend the IAM (Identity and Access Management) feature for the service mesh to enhance the security of the service. When service A accesses service B, it will carry the obtained token. After receiving the request, service B verifies the token through the authentication service, and based on the verification result, decides whether to serve the request or not.

Two plugins are required here: 

- `token-injector` to inject the token into the request from service A
- `token-verifyer` to verify the identity of the request accessing service B. Both of them handle outbound and inbound traffic, respectively.

Corresponding to this are two `PluginChain`s: 

- `token-injector-chain` 
- `token-verifier-chain`

![](/docs/images/plugins/16756463922829.gif)

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have osm-edge installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.

## Deploy demo services

```shell
kubectl create namespace curl
osm namespace add curl
kubectl apply -n curl -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/samples/curl/curl.yaml

kubectl create namespace httpbin
osm namespace add httpbin
kubectl apply -n httpbin -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/samples/httpbin/httpbin.yaml

sleep 2
kubectl wait --for=condition=ready pod -n curl -l app=curl --timeout=90s
kubectl wait --for=condition=ready pod -n httpbin -l app=httpbin --timeout=90s

curl_pod=`kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}'`
httpbin_pod=`kubectl get pod -n httpbin -l app=httpbin -o jsonpath='{.items..metadata.name}'`
```

To view the content of the plugin chains for both services. The **built-in plugins** are located in the `modules` directory. 

> These built-in plugins are native functions provided by the service mesh and are not configured through plugin mechanism, but can be overridden via `plugin` mechanism.

```shell
osm proxy get config_dump -n curl $curl_pod | jq '.Chains."outbound-http"'
[
  "modules/outbound-http-routing.js",
  "modules/outbound-metrics-http.js",
  "modules/outbound-tracing-http.js",
  "modules/outbound-logging-http.js",
  "modules/outbound-circuit-breaker.js",
  "modules/outbound-http-load-balancing.js",
  "modules/outbound-http-default.js"
]

osm proxy get config_dump -n httpbin $httpbin_pod | jq '.Chains."inbound-http"'
[
  "modules/inbound-tls-termination.js",
  "modules/inbound-http-routing.js",
  "modules/inbound-metrics-http.js",
  "modules/inbound-tracing-http.js",
  "modules/inbound-logging-http.js",
  "modules/inbound-throttle-service.js",
  "modules/inbound-throttle-route.js",
  "modules/inbound-http-load-balancing.js",
  "modules/inbound-http-default.js"
]
```

Test communication between the applications.

```shell
kubectl exec $curl_pod -n curl -c curl -- curl -Is http://httpbin.httpbin:14001/get
HTTP/1.1 200 OK
server: gunicorn/19.9.0
date: Sun, 05 Feb 2023 05:42:51 GMT
content-type: application/json
content-length: 304
access-control-allow-origin: *
access-control-allow-credentials: true
connection: keep-alive
```

### Deploy authentication service

Deploy a standalone authentication service to authenticate requests and return `200` or `401`. For simplicity, here we have hard-coded a valid token as `2f1acc6c3a606b082e5eef5e54414ffb`

```shell
kubectl create namespace auth

kubectl apply -n auth -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ext-auth
  name: ext-auth
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ext-auth
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ext-auth
    spec:
      containers:
      - command:
        - pipy
        - -e
        - |2-

          pipy({
            _acceptTokens: ['2f1acc6c3a606b082e5eef5e54414ffb'],
            _allow: false,
          })

            // Pipeline layouts go here, e.g.:
            .listen(8079)
            .demuxHTTP().to($ => $
              .handleMessageStart(
                msg => ((token = msg?.head?.headers?.['x-iam-token']) =>
                  _allow = token && _acceptTokens?.find(el => el == token)
                )()
              )
              .branch(() => _allow, $ => $.replaceMessage(new Message({ status: 200 })),
                $ => $.replaceMessage(new Message({ status: 401 }))
              )
            )
        image: flomesh/pipy:latest
        name: pipy
        resources: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ext-auth
  name: ext-auth
spec:
  ports:
  - port: 8079
    protocol: TCP
    targetPort: 8079
  selector:
    app: ext-auth
EOF
```

### Enable plugin policy mode

To enable plugin policy, the [mesh configuration](/docs/api_reference/config/v1alpha2/) needs to be modified as plugin policies are not enabled by default.

```shell
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enablePluginPolicy":true}}}' --type=merge
```

### Declaring a plugin

Plugin `token-injector`:

- `metadata.name`: the name of the plugin, which is also the name of the plugin script. For example, this plugin will be saved as `token-injector.js` stored in the plugins directory of the code repository.
- `spec.pipyscript`: the [PipyJS](https://flomesh.io/pipy/docs/en/reference/pjs) script content, which is the functional logic code, stored in the script file `plugins/token-injector.js`. Context metadata that is built-in to the system can be used within the script.
- `spec.priority`: the priority of the plugin, with optional values of 0-65535. The higher the value, the higher the priority, and the earlier the plugin is positioned in the plugin chain. The value here is `115`, which, based on the built-in plugin list in Helm [values.yaml](https://github.com/flomesh-io/osm-edge/blob/45b05bd39dc0e8d1c28460622a4be2f92abdf28f/charts/osm/values.yaml#L84), will be positioned between `modules/outbound-circuit-breaker.js` and `modules/outbound-http-load-balancing.js`, executed after the circuit breaker logic is processed and before the load balancer forwards to the upstream.

```shell
kubectl apply -f - <<EOF
kind: Plugin
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: token-injector
spec:
  priority: 115
  pipyscript: |+
    (
    pipy({
      _pluginName: '',
      _pluginConfig: null,
      _accessToken: null,
    })
    .import({
        __service: 'outbound-http-routing',
    })
    .pipeline()
    .onStart(
        () => void (
            _pluginName = __filename.slice(9, -3),
            _pluginConfig = __service?.Plugins?.[_pluginName],
            _accessToken = _pluginConfig?.AccessToken
        )
    )
    .handleMessageStart(
        msg => _accessToken && (msg.head.headers['x-iam-token'] = _accessToken)
    )
    .chain()
    )
EOF
```

Plugin `token-verifier`

```shell
kubectl apply -f - <<EOF
kind: Plugin
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: token-verifier
spec:
  priority: 115
  pipyscript: |+
    (
    pipy({
        _pluginName: '',
        _pluginConfig: null,
        _accessToken: null,
        _valid: false,
    })
    .import({
        __service: 'inbound-http-routing',
    })
    .pipeline()
    .onStart(
        () => void (
            _pluginName = __filename.slice(9, -3),
            _pluginConfig = __service?.Plugins?.[_pluginName],
            _accessToken = _pluginConfig?.AccessToken
        )
    )
    .handleMessageStart(
        msg => _valid = (_accessToken && msg.head.headers['accesstoken'] === _accessToken)
    )
    .branch(
        () => _valid, (
            $ => $.chain()
        ), (
            $ => $.replaceMessage(
            new Message({ status: 403 }, 'token verify failed')
            )
        )
    )
    )
EOF
```

### Setting up plugin-chain

plugin chain `token-injector-chain`:

- `metadata.name`: name of plugin chain resource `token-injector-chain`
- `spec.chains`
    - `name`: name of the plugin chain, one of the 4 plugin chains, here it is outbound-http which is the HTTP protocol processing stage for outbound traffic.
    - `plugins`: list of plugins to be inserted into the plugin chain, here `token-injector` is inserted into the plugin chain.
- `spec.selectors`: target of the plugin chain, using [Kubernetes label selector scheme](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/).
  - `podSelector`: pod selector, selects pods with label `app=curl`.
  - `namespaceSelector`: namespace selector, selects namespaces managed by the mesh, i.e., `openservicemesh.io/monitored-by=osm`.

```shell
kubectl apply -n curl -f - <<EOF
kind: PluginChain
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: token-injector-chain
spec:
  chains:
    - name: outbound-http
      plugins:
        - token-injector
  selectors:
    podSelector:
      matchLabels:
        app: curl
      matchExpressions:
        - key: app
          operator: In
          values: ["curl"]
    namespaceSelector:
      matchExpressions:
        - key: openservicemesh.io/monitored-by
          operator: In
          values: ["osm"]
EOF
```
plugin chain `token-verifier-chain`:

```shell
kubectl apply -n httpbin -f - <<EOF
kind: PluginChain
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: token-verifier-chain
spec:
  chains:
    - name: inbound-http
      plugins:
        - token-verifier
  selectors:
    podSelector:
      matchLabels:
        app: httpbin
    namespaceSelector:
      matchExpressions:
        - key: openservicemesh.io/monitored-by
          operator: In
          values: ["osm"]
EOF
```

After applying the plugin chain configuration, the plugin chains of the two applications can be viewed now. From the results, we can see the two plugins located in the plugins directory. Our declared plugins have been configured in the two applications through the configuration of the plugin chain.

```shell
osm proxy get config_dump -n curl $curl_pod | jq '.Chains."outbound-http"'
[
  "modules/outbound-http-routing.js",
  "modules/outbound-metrics-http.js",
  "modules/outbound-tracing-http.js",
  "modules/outbound-logging-http.js",
  "modules/outbound-circuit-breaker.js",
  "plugins/token-injector.js",
  "modules/outbound-http-load-balancing.js",
  "modules/outbound-http-default.js"
]

osm proxy get config_dump -n httpbin $httpbin_pod | jq '.Chains."inbound-http"'
[
  "modules/inbound-tls-termination.js",
  "modules/inbound-http-routing.js",
  "modules/inbound-metrics-http.js",
  "modules/inbound-tracing-http.js",
  "modules/inbound-logging-http.js",
  "modules/inbound-throttle-service.js",
  "modules/inbound-throttle-route.js",
  "plugins/token-verifier.js",
  "modules/inbound-http-load-balancing.js",
  "modules/inbound-http-default.js"
]
```

After applying the plugin configuration, but we haven't yet make changes to plugin configuration, the application `curl` can still access `httpbin`.

```shell
kubectl exec $curl_pod -n curl -c curl -- curl -Is http://httpbin.httpbin:14001/get
HTTP/1.1 200 OK
server: gunicorn/19.9.0
date: Sun, 05 Feb 2023 06:34:33 GMT
content-type: application/json
content-length: 304
access-control-allow-origin: *
access-control-allow-credentials: true
connection: keep-alive
```

### Setting up plugin configuration

We will first apply the configuration of the plugin `token-verifier`. Here, the authentication service `ext-auth.auth:8079` and the request `/get` that needs to be authenticated are configured.

- `spec.config` contains the contents of the plugin configuration, which will be converted to JSON format. For example, the configuration applied to the `token-verifier` plugin exists in the following JSON form:

  ```json
  {
    "Plugins": {
      "token-verifier": {
        "Paths": [
          "/get"
        ],
        "Verifier": "ext-auth.auth:8079"
      }
    }
  }
  ```

```shell
kubectl apply -n httpbin -f - <<EOF
kind: PluginConfig
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: token-verifier-config
spec:
  config:
    Verifier: 'ext-auth.auth:8079'
    Paths:
      - "/get"
  plugin: token-verifier
  destinationRefs:
    - kind: Service
      name: httpbin
      namespace: httpbin
EOF
```

At this time, the application `curl` cannot access the `httbin` `/get` path because a access token has not been configured for `curl` yet.

```shell
kubectl exec $curl_pod -n curl -c curl -- curl -Is http://httpbin.httpbin:14001/get
HTTP/1.1 401 Unauthorized
content-length: 13
connection: keep-alive
```

But accessing `/headers` path doesn't require any authentication.

```shell
kubectl exec $curl_pod -n curl -c curl -- curl -Is http://httpbin.httpbin:14001/headers
HTTP/1.1 200 OK
server: gunicorn/19.9.0
date: Sun, 05 Feb 2023 06:37:05 GMT
content-type: application/json
content-length: 217
access-control-allow-origin: *
access-control-allow-credentials: true
connection: keep-alive
```

Next, the configuration of the plugin `token-injector` is applied to configure the access token `2f1acc6c3a606b082e5eef5e54414ffb` for the application's requests. This token is also a valid token hardcoded in the authentication service.

```shell
kubectl apply -n curl -f - <<EOF
kind: PluginConfig
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: token-injector-config
spec:
  config:
    AccessToken: '2f1acc6c3a606b082e5eef5e54414ffb'
  plugin: token-injector
  destinationRefs:
    - kind: Service
      name: httpbin
      namespace: httpbin
EOF
```

After applying the configuration for the token-injector plugin, the requests from the `curl` application will now have the access token `2f1acc6c3a606b082e5eef5e54414ffb` configured. As a result, when accessing the `/get` path of `httpbin`, the requests will pass authentication and be accepted by `httpbin`.

```shell
kubectl exec $curl_pod -n curl -c curl -- curl -Is http://httpbin.httpbin:14001/get
HTTP/1.1 200 OK
server: gunicorn/19.9.0
date: Sun, 05 Feb 2023 06:39:54 GMT
content-type: application/json
content-length: 360
access-control-allow-origin: *
access-control-allow-credentials: true
connection: keep-alive
```











