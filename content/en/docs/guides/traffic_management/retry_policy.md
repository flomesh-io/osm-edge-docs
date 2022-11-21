---
title: "Retry"
description: "Impelmenting Retry to handle transient failures"
type: docs
weight: 11
---

# Retry

Retry is a resiliency pattern that enables an application to shield transient issues from customers. This is done by retrying requests that are failing from temporary faults such as a pod is starting up. This guide describes how to implement retry policy in osm-edge.

## Configuring Retry

osm-edge uses its [Retry policy API][1] to allow retries on traffic from a specified source (ServiceAccount) to one or more destinations (Service). Retry is only applicable to HTTP traffic. osm-edge can implement retry for applications participating in the mesh.

The following retry configurations are supported:

- `Per Try Timeout`: The time allowed for a retry to take before it is considered a failed attempt. The default uses the global route timeout.

- `Retry Backoff Base Interval`: The base interval for exponential retry back-off. The backoff is randomly chosen from the range [0,(2**N-1)B], where N is the retry number and B is the base interval. The default is `25ms` and the maximum interval is 10 times the base interval.

- `Number of Retries`: The maximum number of retries to attempt. The default is `1`.

- `Retry On`: Specifies the policy for when a failed request will be retried. Multiple policies can be specified by using a `,` delimited list.

To learn more about configuring retry, refer to the [Retry policy demo](docs/demos/retry_policy) and [API documentation][1].

### Examples
If requests from the bookbuyer service to bookstore-v1 service or bookstore-v2 service receive responses with a status code 5xx, then bookbuyer will retry the request 3 times. If an attempted retry takes longer than 3s it's considered a failed attempt. Each retry has a delay period (backoff) before it is attempted [calculated above](#configuring-retry). The backoff for all retries is capped at 10s.
```yaml
kind: Retry
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: retry
spec:
  source:
    kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
  destinations:
  - kind: Service
    name: bookstore
    namespace: bookstore-v1
  - kind: Service
    name: bookstore
    namespace: bookstore-v2
  retryPolicy:
    retryOn: "5xx"
    perTryTimeout: 3s
    numRetries: 3
    retryBackoffBaseInterval: 1s
```

If requests from the bookbuyer service to bookstore-v2 service receive responses with a status code 5xx or retriable-4xx (409), then bookbuyer will retry the request 5 times. If an attempted retry takes longer than 4s it's considered a failed attempt. Each retry has a delay period (backoff) before it is attempted [calculated above](#configuring-retry). The backoff for all retries is capped at 20ms.
```yaml
kind: Retry
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: retry
spec:
  source:
    kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
  destinations:
  - kind: Service
    name: bookstore
    namespace: bookstore-v2
  retryPolicy:
    retryOn: "5xx,retriable-4xx"
    perTryTimeout: 4s
    numRetries: 5
    retryBackoffBaseInterval: 2ms
```

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.RetrySpec