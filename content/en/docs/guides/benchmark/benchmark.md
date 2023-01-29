---
title: "Data plane benchmarking"
description: "benchmarking osm-edge and osm data planes"
type: docs
weight: 1
---

osm-edge aims to provide service esh functionality along with high performance resources, so that resource-constrained edge environments can also use the service mesh functionality used in the cloud.

In this test, benchmarks were conducted for osm-edge (v1.1.0) and osm (v1.1.0). The main focus is on service TPS, latency distribution when using two different meshes, and monitoring the resource overhead of the data plane.

osm-edge uses Pipy as the data plane; osm uses Envoy as the data plane.

## Testing Environment

The benchmark was tested in a Kubernetes cluster running on Tencent Cloud CVM. There are 2 standard S5 nodes in the cluster. osm-edge and osm are both **on loose traffic mode and mTLS, and the other settings are default**.

* Kubernetes: k3s v1.20.14+k3s2
* OS: Ubuntu 20.04
* Nodes: 16c32g * 2
* Load Generator: 8c16g

The test application uses the common SpringCloud microservices architecture. The application is taken from [flomesh-bookinfo-demo](https://github.com/flomesh-io/flomesh-bookinfo-demo/), which is a SpringCloud implementation of bookinfo application using SpringCloud. In the tests, not all services are used, but rather the API gateway and the Bookinfo Ratings service are selected.

![](https://user-images.githubusercontent.com/2224492/178288704-3aa44151-4c57-4538-9a0a-55310bb4f200.png)

External access to the service is provided through Ingress; the load generator uses the common Apache Jmeter 5.5.

For the tests, 2 links were chosen: one is the direct access to the Bookinfo Ratings service via Ingress (hereafter ratings), and the other passes through the API Gateway (hereafter gateway-ratings) in the middle. The reason for choosing these two links is to cover both single sidecar and multiple sidearm scenarios.

## Procedure

We start the test with a non-mesh (i.e., no sidecar injection, hereafter referred to as non-mesh), and then test with osm-edge and osm mesh, respectively.

In order to simulate the resource-constrained scenario of the edge, the CPU used by the sidecar is limited when using the grid, and the 1 core and 2 core scenarios are tested respectively.

Thus, there are 5 rounds of testing, with 2 different links in each round.

## Performance

Jmeter uses 200 threads during the test, and runs the test for 5 minutes. Each round of testing is preceded by a 2-minute warm-up.

For space reasons, only the results of the gateway-ratings link are shown below with different sidecar resource constraints.

*Note: The sidecar in the table refers to the API Gateway sidecar*.

| grid | sidecar max CPU | TPS | 90th | 95th | 99th | sidecar CPU usage | sidecar memory usage |
|----------|:-----------------|------|:-----|:-----|:-----|:-----------------|:----------------|
| non-mesh | NA | 3283 | 87 | 89 | 97 | NA | NA | NA |
| osm-edge | 2 | 3395 | 77 | 79 | 84 | 130% | 52 |
| osm | 2 | 2189 | 102 | 104 | 109 | 200% | 108 |
| osm-edge | 1 | 2839 | 76 | 77 | 79 | 100% | 34 |
| The osm-edge | 1 | 1097 | 201 | 203 | 285 | 100% | 105 |


### sidecar 2 core

In the sidecar 2-core scenario, using the osm-edge grid gives a small TPS improvement over not using the grid, and also improves latency. The sidecar for both API Gateway and Bookinfo Ratings is still not running out of 2 cores (only 65%), when the Bookinfo Ratings service itself is at its performance limit.

The TPS of the osm grid was down nearly 30%, and the API Gateway sidecar CPU was running full, which was the bottleneck.

In terms of memory, the memory usage of osm-edge and osm sidecar is 50 MiB and 105 MiB respectively.

**TPS**

![](https://user-images.githubusercontent.com/2224492/178294418-d3d63aef-8c54-49e4-a40e-8bacdec26f74.png)

**Latency distribution**

![](https://user-images.githubusercontent.com/2224492/178294471-b8e1b3c6-a8fd-47cb-872a-0c40418b0da7.png)

**API gateway sidecar CPU usage**

![](https://user-images.githubusercontent.com/2224492/178294732-73aaa9f4-e159-4b8e-ab12-521985313358.png)

**API gateway sidecar memory footprint**

![](https://user-images.githubusercontent.com/2224492/178294829-9d2f0794-12e7-4cd6-827d-11af8b632db9.png)

**Bookinfo Ratings sidecar CPU usage**

![](https://user-images.githubusercontent.com/2224492/178295086-6380004f-369d-4f6b-afeb-71b48c0e3053.png)

**Bookinfo Ratings sidecar memory usage**

![](https://user-images.githubusercontent.com/2224492/178295267-004e7676-04b5-4fef-8e5d-ca196bd7dedc.png)

### Sidecar 1 core

The difference is particularly noticeable in tests that limit sidecar to 1 core CPU. At this point, the API Gateway sidecar becomes the performance bottleneck, with both the osm-edge and osm sidecar running out of CPU.

In terms of TPS, osm-edge drops 12% and osm's TPS drops a staggering 65%.

**TPS**!

![](https://user-images.githubusercontent.com/2224492/178295573-8be92413-d499-476e-b3e1-a23d0bcbcda3.png)

**Latency distribution**

![](https://user-images.githubusercontent.com/2224492/178296728-c7ea9a12-d9d4-4be0-9c8d-32bb91724f36.png)

**API gateway sidecar CPU usage**

![](https://user-images.githubusercontent.com/2224492/178300176-0a76080b-3bcb-48f4-a506-4a105ad8c4a8.png)

**API gateway sidecar memory footprint**

![](https://user-images.githubusercontent.com/2224492/178300241-95917e41-5857-4a80-8234-ff6533310ef5.png)

**Bookinfo Ratings sidecar CPU usage**

![](https://user-images.githubusercontent.com/2224492/178300596-2f767c75-6872-4aa5-b943-a3eeae84c55e.png)

**Bookinfo Ratings sidecar memory usage**

![](https://user-images.githubusercontent.com/2224492/178300658-53bdf00d-6f2f-484a-8c3f-0399f9b683ed.png)

## Summary

This time, we benchmarked osm-edge and osm data planes with limited sidecar resources. From the results, osm-edge can still maintain high performance with low resource usage and more efficient use of resources. For resource-constrained edge scenarios, service grid features that are only available in the cloud can be enjoyed at a lower resource overhead. These are made possible by [Pipy](https://flomesh.io)'s low-resource, high-performance features.

Of course, osm-edge is suitable for edge computing scenarios, but it can be applied to the cloud as well. In particular, cloud environments with large-scale services meet the requirements for cost control.