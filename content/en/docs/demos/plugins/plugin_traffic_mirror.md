---
title: "Traffic Mirroring"
description: "Adding Traffic Shadowing functionality to osm-edge sidecar via Plugins"
type: docs
weight: 4
---

Traffic mirroring, also known as shadowing, is a technique used to test new versions of an application in a safe and efficient manner. It involves creating a mirrored service that receives a copy of live traffic for testing and troubleshooting purposes. This approach is especially useful for acceptance testing, as it can help identify issues in advance, before they impact end-users.

One of the key benefits of traffic mirroring is that it occurs outside the primary request path for the main service. This means that end-users are not affected by any changes or issues that may occur during the testing process. As such, traffic mirroring is a powerful and low-risk approach for validating new versions of an application.

By using traffic mirroring, you can get valuable insights into how your application will perform in a live environment, without putting your users at risk. This approach can help you identify and address issues quickly and efficiently, which can ultimately improve the overall performance and reliability of your application.

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
      - image: flomesh/httpbin:ken
        imagePullPolicy: IfNotPresent
        name: pipy
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:8080", "httpbin:app"]
        ports:
        - containerPort: 8080
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
      - image: flomesh/httpbin:ken
        imagePullPolicy: IfNotPresent
        name: pipy
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:8080", "httpbin:app"]
        ports:
        - containerPort: 8080
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
  name: traffic-mirror
spec:
  priority: 115
  pipyscript: |+
    ((
      config = pipy.solve('config.js'),
      clusterCache = new algo.Cache(
        (clusterName => (
          (cluster = config?.Outbound?.ClustersConfigs?.[clusterName]) => (
            cluster ? Object.assign({ name: clusterName }, cluster) : null
          )
        )())
      ),
      hexChar = { '0': 0, '1': 1, '2': 2, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9, 'a': 10, 'b': 11, 'c': 12, 'd': 13, 'e': 14, 'f': 15 },
      randomInt63 = () => (
        algo.uuid().substring(0, 18).replaceAll('-', '').split('').reduce((calc, char) => (calc * 16) + hexChar[char], 0) / 2
      ),
      samplingRange = fraction => (fraction > 0 ? fraction : 0) * Math.pow(2, 63),
      configCache = new algo.Cache(
        pluginConfig => pluginConfig && (
          {
            samplingRange: pluginConfig?.percentage?.value > 0 ? samplingRange(pluginConfig.percentage.value) : 0,
            clusterName: pluginConfig?.namespace + '/' + pluginConfig?.service + '|' + pluginConfig?.port,
            namespace: pluginConfig?.namespace,
            service: pluginConfig?.service,
            port: pluginConfig?.port,
          }
        )
      ),
    ) => pipy({
      _pluginName: '',
      _pluginConfig: null,
      _mirrorConfig: null,
      _randomVal: 0,
      _mirrorCluster: undefined,
    })
    .import({
      __service: 'outbound-http-routing',
      __cluster: 'outbound-http-routing',
    })
    .pipeline()
    .onStart(
      () => void (
        _pluginName = __filename.slice(9, -3),
        _pluginConfig = __service?.Plugins?.[_pluginName],
        (_mirrorConfig = configCache.get(_pluginConfig)) && (
          _mirrorCluster = clusterCache.get(_mirrorConfig.clusterName)
        )
      )
    )
    .handleMessageStart(
      () => (
        _mirrorCluster && (
          _randomVal = randomInt63(),
          (_randomVal < _mirrorConfig.samplingRange) || (
            _mirrorCluster = undefined
          )
        )
      )
    )
    .branch(
      () => _mirrorCluster, (
        $=>$
        .fork().to('mirror-cluster')
        .chain()
      ), (
        $=>$.chain()
      )
    )
    
    .pipeline('mirror-cluster')
    .replaceMessage(
      msg => (
        (
          mirrorMsg = new Message(Object.assign({}, msg.head), msg.body),
          hostParts = msg.head.headers.host.split('.'),
        ) => (
          __cluster = _mirrorCluster,
          hostParts?.length > 0 && (
            hostParts[0] = _mirrorConfig.service,
            mirrorMsg.head.headers = Object.assign({}, msg.head.headers),
            mirrorMsg.head.headers.host = hostParts.join('.')
          ),
          mirrorMsg
        )
      )()
    )
    .chain([
      'modules/outbound-http-load-balancing.js',
      'modules/outbound-http-default.js',
    ])
    .dummy()
    )()
EOF
```

### Setting up plugin-chain

```bash
kubectl apply -f - <<EOF
kind: PluginChain
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: traffic-mirror-chain
  namespace: pipy
spec:
  chains:
    - name: outbound-http
      plugins:
        - traffic-mirror
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

### Setting up plugin configuration

```bash
kubectl apply -f - <<EOF
kind: PluginConfig
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: traffic-mirror-config
  namespace: curl
spec:
  config:
    namespace: pipy
    service: pipy-ok-v2
    port: 8080
    percentage:
      value: 1.0
  plugin: traffic-mirror
  destinationRefs:
    - kind: Service
      name: pipy-ok-v1
      namespace: pipy
EOF
```

## Test

Use the below command to perform a test

```bash
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n curl -c curl -- curl -ksi http://pipy-ok-v1.pipy:8080 
```

Accessing service `pipy-ok-v1` should be mirrored to `pipy-ok-v2`, so we should be seeing access logs in both services.

### Testing pipy-ok-v1 logs

```bash
pipy_ok_v1="$(kubectl get pod -n pipy -l app=pipy-ok,version=v1 -o jsonpath='{.items[0].metadata.name}')"
kubectl logs pod/${pipy_ok_v1} -n pipy -c pipy
```

You will see something similar

```console
[2023-03-29 08:41:04 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2023-03-29 08:41:04 +0000] [1] [INFO] Listening at: http://0.0.0.0:8080 (1)
[2023-03-29 08:41:04 +0000] [1] [INFO] Using worker: sync
[2023-03-29 08:41:04 +0000] [14] [INFO] Booting worker with pid: 14
127.0.0.6 - - [29/Mar/2023:08:45:35 +0000] "GET / HTTP/1.1" 200 9593 "-" "curl/7.85.0-DEV"
```

### Testing pipy-ok-v2 logs

```bash
pipy_ok_v2="$(kubectl get pod -n pipy -l app=pipy-ok,version=v2 -o jsonpath='{.items[0].metadata.name}')"
kubectl logs pod/${pipy_ok_v2} -n pipy -c pipy
```

You will see something similar

```console
[2023-03-29 08:41:09 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2023-03-29 08:41:09 +0000] [1] [INFO] Listening at: http://0.0.0.0:8080 (1)
[2023-03-29 08:41:09 +0000] [1] [INFO] Using worker: sync
[2023-03-29 08:41:09 +0000] [15] [INFO] Booting worker with pid: 15
127.0.0.6 - - [29/Mar/2023:08:45:35 +0000] "GET / HTTP/1.1" 200 9593 "-" "curl/7.85.0-DEV"
```