---
title: "Fault Injection"
description: "Adding Fault Injection functionality to osm-edge sidecar via Plugins"
type: docs
weight: 3
---

Fault injection testing is a software testing technique that intentionally introduces errors into a system to verify its ability to handle and bounce back from error conditions. This testing method is usually performed before deployment to identify any possible faults that may have arisen during production. This demo demonstrates on how to implement Fault Injection functionality via a plugin.

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
  name: http-fault-injection
spec:
  priority: 165
  pipyscript: |+
    ((
      seconds = val => (
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
      hexChar = { '0': 0, '1': 1, '2': 2, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9, 'a': 10, 'b': 11, 'c': 12, 'd': 13, 'e': 14, 'f': 15 },    
      randomInt63 = () => (
        algo.uuid().substring(0, 18).replaceAll('-', '').split('').reduce((calc, char) => (calc * 16) + hexChar[char], 0) / 2
      ),    
      samplingRange = fraction => (fraction > 0 ? fraction : 0) * Math.pow(2, 63),    
      configCache = new algo.Cache(
        pluginConfig => pluginConfig && (
          {
            delaySamplingRange: pluginConfig?.delay?.percentage?.value > 0 ? samplingRange(pluginConfig.delay.percentage.value) : 0,
            fixedDelay: seconds(pluginConfig?.delay?.fixedDelay),
            abortSamplingRange: pluginConfig?.abort?.percentage?.value > 0 && pluginConfig?.abort?.httpStatus > 0 ? (
              samplingRange(pluginConfig.abort.percentage.value)
            ) : 0,
            httpStatus: pluginConfig?.abort?.httpStatus,
          }
        )
      ),      
    ) => pipy({
      _pluginName: '',
      _pluginConfig: null,
      _faultConfig: null,
      _randomVal: 0,
      _delayFlag: false,
      _abortFlag: false,
    })
    .import({
      __service: 'inbound-http-routing',
    })
    .pipeline()
    .onStart(
      () => void (
        _pluginName = __filename.slice(9, -3),
        _pluginConfig = __service?.Plugins?.[_pluginName],
        _faultConfig = configCache.get(_pluginConfig)
      )
    )
    .handleMessageStart(
      () => (
        _faultConfig && (
          _randomVal = randomInt63(),
          _faultConfig.delaySamplingRange && (_randomVal < _faultConfig.delaySamplingRange) && (     
            _delayFlag = true
          ),
          _faultConfig.abortSamplingRange && (_randomVal < _faultConfig.abortSamplingRange) && (
            _abortFlag = true
          )
        )
      )
    )
    .branch(
      () => _delayFlag, (
        $=>$.replay({ delay: () => _faultConfig.fixedDelay }).to(
          $=>$.branch(
            () => _delayFlag && (_delayFlag = false, true), (
              $=>$.replaceMessageStart(
                () => new StreamEnd('Replay')
              )
            ), (
              $=>$
            )
          )
        )
      ), (
        $=>$
      )
    )
    .branch(
      () => _abortFlag, (
        $=>$.replaceMessage(
          () => (
            new Message({ status: _faultConfig.httpStatus })
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
  name: http-fault-injection-chain
  namespace: pipy
spec:
  chains:
    - name: inbound-http
      plugins:
        - http-fault-injection
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

In below configuration, we need to have either of `delay` or `abort`

```bash
kubectl apply -f - <<EOF
kind: PluginConfig
apiVersion: plugin.flomesh.io/v1alpha1
metadata:
  name: http-fault-injection-config
  namespace: pipy
spec:
  config:
    delay:
      percentage:
        value: 0.5
      fixedDelay: 5s
    abort:
      percentage:
        value: 0.5
      httpStatus: 400
  plugin: http-fault-injection
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

date;  kubectl exec ${curl_client} -n curl -c curl -- curl -ksi http://pipy-ok.pipy:8080 ;  echo "";  date
```

Run the above command a few times, and you will see that after a few more visits, and there is  approximately a 50% chance of receiving the following result (HTTP status code 400, with a delay of 5 seconds).

```bash
Thu Mar 30 06:47:58 UTC 2023
HTTP/1.1 400 Bad Request
content-length: 0
connection: keep-alive


Thu Mar 30 06:48:04 UTC 2023
```


