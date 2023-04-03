---
title: "Cross-Origin Resource Sharing (CORS)"
description: "Adding CORS functionality to osm-edge sidecar via Plugins"
type: docs
weight: 5
---

CORS stands for Cross-Origin Resource Sharing. It is a security feature implemented by web browsers to prevent web pages from making requests to a different domain than the one that served the web page.

The same-origin policy is a security feature that allows web pages to access resources only from the same origin, which includes the same domain, protocol, and port number. This policy is designed to protect users from malicious scripts that can steal sensitive data from other websites.

CORS allows web pages to make cross-origin requests by adding specific headers to the HTTP response from the server. The headers indicate which domains are allowed to make cross-origin requests, and what type of requests are allowed.

However, configuring CORS can be challenging, especially when dealing with complex web applications that involve multiple domains and servers. One way to simplify the CORS configuration is to use a proxy server.

A proxy server acts as an intermediary between the web application and the server. The web application sends requests to the proxy server, which then forwards the requests to the server. The server responds to the proxy server, which then sends the response back to the web application.

By using a proxy server, you can configure the CORS headers on the proxy server instead of configuring them on the server that serves the web page. This way, the web application can make cross-origin requests without violating the same-origin policy.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have osm-edge installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.

## Deploy demo services

```bash
kubectl create namespace curl
osm namespace add curl
kubectl create namespace pipy
osm namespace add pipy

kubectl apply -n curl -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: curl
---
apiVersion: v1
kind: Service
metadata:
  name: curl
  labels:
    app: curl
    service: curl
spec:
  ports:
    - name: http
      port: 80
  selector:
    app: curl
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      serviceAccountName: curl
      containers:
      - image: curlimages/curl:latest
        imagePullPolicy: IfNotPresent
        name: curl
        command: ["sleep", "365d"]
EOF

kubectl apply -n pipy -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipy-ok-v1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipy-ok-v2
---
apiVersion: v1
kind: Service
metadata:
  name: pipy-ok
spec:
  ports:
    - name: pipy
      port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy-ok
---
apiVersion: v1
kind: Service
metadata:
  name: pipy-ok-v1
spec:
  ports:
    - name: pipy
      port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy-ok
    version: v1
---
apiVersion: v1
kind: Service
metadata:
  name: pipy-ok-v2
spec:
  ports:
    - name: pipy
      port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy-ok
    version: v2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipy-ok-v1
  labels:
    app: pipy-ok
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy-ok
      version: v1
  template:
    metadata:
      labels:
        app: pipy-ok
        version: v1
    spec:
      serviceAccountName: pipy-ok-v1
      containers:
        - name: pipy
          image: flomesh/pipy:latest
          ports:
            - name: pipy
              containerPort: 8080
          command:
            - pipy
            - -e
            - |
              pipy()
              .listen(8080)
              .serveHTTP(new Message('Hi, I am PIPY-OK v1!'))
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipy-ok-v2
  labels:
    app: pipy-ok
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy-ok
      version: v2
  template:
    metadata:
      labels:
        app: pipy-ok
        version: v2
    spec:
      serviceAccountName: pipy-ok-v2
      containers:
        - name: pipy
          image: flomesh/pipy:latest
          ports:
            - name: pipy
              containerPort: 8080
          command:
            - pipy
            - -e
            - |
              pipy()
              .listen(8080)
              .serveHTTP(new Message('Hi, I am PIPY-OK v2!'))
EOF

# Wait for pods to be up and ready

sleep 2
kubectl wait --for=condition=ready pod -n curl -l app=curl --timeout=180s
kubectl wait --for=condition=ready pod -n pipy -l app=pipy-ok -l version=v1 --timeout=180s
kubectl wait --for=condition=ready pod -n pipy -l app=pipy-ok -l version=v2 --timeout=180s
```

### Enable plugin policy mode

```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enablePluginPolicy":true}}}' --type=merge
```

### Declaring a plugin

```bash
kubectl apply -f - <<EOF
kind: Plugin
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: cors-policy
spec:
  priority: 165
  pipyscript: |+
    ((
      cacheTTL = val => (
        val?.indexOf('s') > 0 && (
        val.replace('s', '')
        ) ||
        val?.indexOf('m') > 0 && (
        val.replace('m', '') * 60
        ) ||
        val?.indexOf('h') > 0 && (
        val.replace('h', '') * 3600
        ) ||
        val?.indexOf('d') > 0 && (
        val.replace('d', '') * 86400
        ) ||
        0
      ),
      originMatch = origin => (
        (origin || []).map(
        o => (
          o?.exact && (
            url => url === o.exact
          ) ||
          o?.prefix && (
            url => url.startsWith(o.prefix)
          ) ||
          o?.regex && (
            (match = new RegExp(o.regex)) => (
            url => match.test(url)
            )
          )()
        )
        )
      ),
      configCache = new algo.Cache(
        pluginConfig => (
        (originHeaders = {}, optionsHeaders = {}) => (
          pluginConfig?.allowCredentials && (
            originHeaders['access-control-allow-credentials'] = 'true'
          ),
          pluginConfig?.exposeHeaders && (
            originHeaders['access-control-expose-headers'] = pluginConfig.exposeHeaders.join()
          ),
          pluginConfig?.allowMethods && (
            optionsHeaders['access-control-allow-methods'] = pluginConfig.allowMethods.join()
          ),
          pluginConfig?.allowHeaders && (
            optionsHeaders['access-control-allow-headers'] = pluginConfig.allowHeaders.join()
          ),
          pluginConfig?.maxAge && (cacheTTL(pluginConfig?.maxAge) > 0) && (
            optionsHeaders['access-control-max-age'] = cacheTTL(pluginConfig?.maxAge)
          ),
          {
            originHeaders,
            optionsHeaders,
            matchingMap: originMatch(pluginConfig?.allowOrigins)
          }
        )
        )()
      ),
    ) => pipy({
      _pluginName: '',
      _pluginConfig: null,
      _corsHeaders: null,
      _matchingMap: null,
      _matching: false,
      _isOptions: false,
      _origin: undefined,
    })
    .import({
      __service: 'inbound-http-routing',
    })
    .pipeline()
    .onStart(
      () => void (
        _pluginName = __filename.slice(9, -3),
        _pluginConfig = __service?.Plugins?.[_pluginName],
        _corsHeaders = configCache.get(_pluginConfig),
        _matchingMap = _corsHeaders?.matchingMap
      )
    )
    .branch(
      () => _matchingMap, (
        $=>$
        .handleMessageStart(
        msg => (
          (_origin = msg?.head?.headers?.origin) && (_matching = _matchingMap.find(o => o(_origin))) && (
            _isOptions = (msg?.head?.method === 'OPTIONS')
          )
        )
        )
      ), (
        $=>$
      )
    )
    .branch(
      () => _matching, (
        $=>$.branch(
        () => _isOptions, (
          $=>$.replaceMessage(
            () => (
            new Message({ status: 200, headers: { ..._corsHeaders.originHeaders, ..._corsHeaders.optionsHeaders, 'access-control-allow-origin': _origin } })
            )
          )
        ), (
          $=>$
          .chain()
          .handleMessageStart(
            msg => (
            Object.keys(_corsHeaders.originHeaders).forEach(
              key => msg.head.headers[key] = _corsHeaders.originHeaders[key]
            ),
            msg.head.headers['access-control-allow-origin'] = _origin
            )
          )
        )
        )
      ), (
        $=>$.chain()
      )
    )
    )()
EOF
```

### Setting up plugin-chain

```bash
kubectl apply -f - <<EOF
kind: PluginChain
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: cors-policy-chain
  namespace: pipy
spec:
  chains:
    - name: inbound-http
      plugins:
        - cors-policy
  selectors:
    podSelector:
      matchLabels:
        app: pipy-ok
      matchExpressions:
        - key: app
          operator: In
          values: ["pipy-ok"]
    namespaceSelector:
      matchExpressions:
        - key: openservicemesh.io/monitored-by
          operator: In
          values: ["osm"]
EOF
```

### Setting up plugin configuration

```bash
kubectl apply -f - <<EOF
kind: PluginConfig
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: cors-policy-config
  namespace: pipy
spec:
  config:
    allowCredentials: true
    allowHeaders:
    - X-Foo-Bar-1
    allowMethods:
    - POST
    - GET
    - PATCH
    - DELETE
    allowOrigins:
    - regex: http.*://www.test.cn
    - exact: http://www.aaa.com
    - prefix: http://www.bbb.com
    exposeHeaders:
    - Content-Encoding
    - Kuma-Revision
    maxAge: 24h
  plugin: cors-policy
  destinationRefs:
    - kind: Service
      name: pipy-ok
      namespace: pipy
EOF
```

## Test

Use the below command to perform a test

```bash
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n curl -c curl -- curl -ksi http://pipy-ok.pipy:8080 -H "Origin: http://www.bbb.com"
```

You will see response similar to:

```console
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-expose-headers: Content-Encoding,Kuma-Revision
access-control-allow-origin: http://www.bbb.com
content-length: 20
connection: keep-alive

Hi, I am PIPY-OK v1!
```


Run another command to perform a test

```bash
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n curl -c curl -- curl -ksi http://pipy-ok.pipy:8080 -H "Origin: http://www.bbb.com" -X OPTIONS
```

You will see response similar to:

```console
HTTP/1.1 200 OK
access-control-allow-origin: http://www.bbb.com
access-control-allow-credentials: true
access-control-expose-headers: Content-Encoding,Kuma-Revision
access-control-allow-methods: POST,GET,PATCH,DELETE
access-control-allow-headers: X-Foo-Bar-1
access-control-max-age: 86400
content-length: 0
connection: keep-alive
```