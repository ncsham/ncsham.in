+++
date = '2025-04-01T11:10:05+05:30'
title = 'K8s Scaling based on Custom Prometheus Scraped Metrics'
tags = ['prometheus', 'adapter', 'hpa', 'kubernetes']
+++

## Introduction
In K8s, we normally do horizontal scaling decisions via the help of HPA manifest i.e Horizontal Pod AutoScaling. 

Inorder to first implement HPA, we need to have metrics which can be seen by the HPA and then do scaling decisions. Normally People install [metrics-server](https://github.com/kubernetes-sigs/metrics-server?tab=readme-ov-file) at the initial creation of cluster. metrics-server exposes the CPU, Mem metrics of pods and nodes directly without any additional configuration other than just applying the metrics-server manifest. It really eases out making scaling decisions based on CPU and MEM with most minimal work.

Now there comes a case where you want to take scaling decisions just not on CPU and MEM but on your own custom metrics like prometheus scraped metrics or some external queue metrics.

In Order for HPA Controller to Do Scaling Decisions based on the custom prometheus metrics, this is where prometheus-adapter comes into place. 

The Whole Document will explain how to do POC for the prometheus Adapter(in Minikube) and also deploy a test application to test the scaling behaviour based on a custom prometheus metrics

## Installation
> NOTE: Before Installing the Prometheus Adapter, Please Make Sure you have a working prometheus Installation in k8s. Default Standard of monitoring the k8s cluster is via the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack). For Installing bare minimum prometheus stack, please refer to the [Testing Section](#testing)

> NOTE: The Whole Installation and Testing Process is being done in Minikube. That's why no namespace argument is not added in any helm commands. But in production we can install the chart in a specific namespace by mentioning `-n <namespace>` cmd line argument to `kubectl`

- Adapter Installation Can be done via the Helm
- Steps
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm show values prometheus-community/prometheus-adapter > values.yaml # To get the default values
```
> NOTE: Now Inorder to See what all manifests the helm chart will install, have run `helm template prometheus-adapter prometheus-community/prometheus-adapter --values values.yaml` command, and cross verify.

- Have kept `.rules.default` as `false` in values.yaml, since by default prometheus adapter is exposing too many metrics, since this is POC, we will be exposing the metrics we wanted.
- Changed the `.prometheus.url` to `http://kps-kube-prometheus-stack-prometheus.default.svc` since kube-prometheus-stack without any configuration will be running at `http://kps-kube-prometheus-stack-prometheus.default.svc:9090`. Change this to where prometheus is located at in custom installations.

```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter --values values.yaml
```

With Chart Successfully installed, we should now be able to see the adapter pod running. We can check the logs to confirm if everything is working fine.

The Chart Basically introduces a new API at kubernetes via the `APIService` Kind Manifest(installed via helm). The API is `custom.metrics.k8s.io`. This is where prometheus adapter exposes the custom metrics.

## Testing
### Install kube-prometheus-stack
For Installing the kube-prometheus-stack in Minikube, we can ran below commands

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kps prometheus-community/kube-prometheus-stack
```
Once chart is installed, we will have basic components like node-exporter, kube-state-metrics, prometheus, alertmanger and grafana installed. After succesful installation of chart, we can see how to access the grafana if required over the output of the install command.

If we want to check the Prometheus UI, we can do port-forwarding i.e `kubectl port-forward svc/kps-kube-prometheus-stack-prometheus 20000:9090` and open `localhost:20000` in browser.

Now that we have Prometheus and Adapter Installed, Its time to Test the Adapter.

First we will install a simple application, which exposes a metric called `http_requests_total` & `http_request_duration_seconds`. Then we can change the configuration of the adapter to expose this metric.

Basic Golang HTTP Server used for Reference was: https://github.com/ncsham/httpserver
### Install Simple Golang HTTP Server
```bash
git clone https://github.com/ncsham/httpserver.git
eval $(minikube docker-env) # In Order to Port Image to Minikube
docker build -t httpserver .
kubectl apply -f k8s.yaml
```

`k8s.yaml` contains Deployment, Service and ServiceMonitor so prometheus can start scraping the metrics. 
> NOTE: we need to add `release: kps` for prometheus-operator to add it as a target in prometheus scrape config. Adding this label can be avoided by setting `serviceMonitorSelector` to `{}` in kube-prometheus-stack helm chart.

The Simple HTTP Server Only Exposes /status endpoint i.e exposes how many times /status was hit and also how many other endpoints were hit. 

We can now start sending traffic to our application and check if the metrics are getting updated via the Prometheus UI. For Sending Traffic, we are gonna use [Hey Load Testing Tool](https://github.com/rakyll/hey). 

### Confirming Metrics at Prometheus
Cross Confirm Whether Metrics are Getting Updated via the Prometheus

  ```bash
  kubectl port-forward svc/gohttpserver 20000:20000 # Port Forward the HTTP Server on localhost, so can start sending traffic via hey
  ```
  
  CrossConfirm Whether Application is working fine
  ```bash
  `--> curl localhost:20000/status && echo
  HTTP/1.1 200 OK
  X-Request-Id: cvlors6e37rs73aqmla0
  Date: Tue, 01 Apr 2025 06:55:44 GMT
  Content-Length: 43
  Content-Type: text/plain; charset=utf-8

  Served with RequestID: cvlors6e37rs73aqmla0

  `--> curl localhost:20000/statu && echo
  HTTP/1.1 404 Not Found
  Date: Tue, 01 Apr 2025 06:57:52 GMT
  Content-Length: 14
  Content-Type: text/plain; charset=utf-8

  Dont Try Again
  ```

  Now in Prometheus UI can See metrics value as expected
  ```
  http_requests_total{code="200", endpoint="metrics", instance="10.244.0.51:20001", job="gohttpserver", method="GET", namespace="default", path="/status", pod="gohttpserver-9f49b6c45-8d2cm", service="gohttpserver"} 1
  http_requests_total{code="404", endpoint="metrics", instance="10.244.0.51:20001", job="gohttpserver", method="GET", namespace="default", path="NA", pod="gohttpserver-9f49b6c45-8d2cm", service="gohttpserver"}	1
  ```

### Configuring Prometheus Adapter
Now Lets Update Prometheus Adapter Configuration to Start Exposing http_requests_total metric at k8s API i.e at `/apis/custom.metrics.k8s.io/v1beta1/`
> NOTE: Since we have kept `.rules.default` as false when installing, intially when `kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/" | jq .` it should give emtpy items[] in response

We need to add below content in `.rules.custom`
```
  - metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)
    name:
      as: ${1}_per_second
      matches: ^(.*)_total
    resources:
      overrides:
        pod: {resource: "pod"}
        namespace: {resource: "namespace"}
    seriesQuery: http_requests_total{job="gohttpserver"}
```

For More Details on How configuration works, Please refer to [Configuration](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md)

Briefly We can say
  - `seriesQuery` is the prometheus query done by adapter to get the metrics series
  - `resources` block tells under which k8s api, the metric should be exposed i.e pod or namespace etc. We can overrides block to expose custom label as resource under the custom metrics api
    - Above Configuration says that pod label value should be exposed under `pods` resource of custom metrics API i.e `/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second`
  - `name` tells what should be the name of the metric when exposing over k8s API i.e `http_requests_per_seconds`
  - `metricsQuery` tells what query to make to prometheus when a k8s api endpoint is called i.e in 2 point above. 
    > NOTE: we have taken rate[1m] in the metrics query

Above can be little confusing at the start. But once we start adding the rules, it will make more sense.

Lets Query the API to Now see whether the metrics are getting exposed when we made one HTTP Request above.
```bash
`--> kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second" | jq .
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "gohttpserver-9f49b6c45-nm5gd",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2025-04-01T07:33:15Z",
      "value": "0",
      "selector": null
    }
  ]
}
```
We can see that k8s api started exposing the metrics as expected at custom.metrics.k8s.io, but the value seems to be zero. Lets try to send 1000 requests and try to validate. 

In order to make it easy to monitor whats happening i.e when metric is getting scraped, when adapter is seeing the change. Have ran below commands in parallel in different terminals

- `watch -n1 'kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second" | jq .` 
- `kubectl port-forward svc/gohttpserver 20000:20000`
- `date && hey -n 1000 -m GET http://localhost:20000/status && date`

```
`--> date && hey -n 1000 -m GET http://localhost:20000/status && date
Tue  1 Apr 2025 13:24:25 IST

Summary:
  Total:	0.3309 secs
  Slowest:	0.1489 secs
  Fastest:	0.0004 secs
  Average:	0.0163 secs
  Requests/sec:	3022.3579

  Total data:	43000 bytes
  Size/request:	43 bytes

Response time histogram:
  0.000 [1]	|
  0.015 [848]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.030 [1]	|
  0.045 [0]	|
  0.060 [50]	|■■
  0.075 [50]	|■■
  0.090 [0]	|
  0.104 [0]	|
  0.119 [1]	|
  0.134 [0]	|
  0.149 [49]	|■■


Latency distribution:
  10% in 0.0019 secs
  25% in 0.0025 secs
  50% in 0.0042 secs
  75% in 0.0055 secs
  90% in 0.0632 secs
  95% in 0.1177 secs
  99% in 0.1469 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0003 secs, 0.0004 secs, 0.1489 secs
  DNS-lookup:	0.0001 secs, 0.0000 secs, 0.0014 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0006 secs
  resp wait:	0.0160 secs, 0.0004 secs, 0.1424 secs
  resp read:	0.0000 secs, 0.0000 secs, 0.0006 secs

Status code distribution:
  [200]	1000 responses



Tue  1 Apr 2025 13:24:25 IST
```
- Initial Metric Value Change at `2025-04-01T07:54:35Z` i.e observed over watch command output
    ```
    "timestamp": "2025-04-01T07:54:35Z"
    "value": "33333m"
    ```
- Metric Value Changed back to 0 at `2025-04-01T07:55:05Z`
    ```
    "timestamp": "2025-04-01T07:55:05Z"
    "value": "0"
    ```
> NOTE: In between, every second the value keeps changing.
### Observations
  - `2025-04-01 07:54:25` -> 1000 Requests were made
  - `2025-04-01 07:54:35` -> Initial Prometheus Scrape Timestamp 
    - At the same time K8s API started exposing the changed value
    - Why value is 33.33RPS? i.e 1000/30 = 33.33 
    > NOTE: We are dividing by 30 but not 60, when we are taking rate over 1m. This is due to scrape interval being 30s & when prometheus gets first and last datapoint for increase, they will be 30s apart.
  - `2025-04-01 07:55:05` -> The Metric Value dropped back to zero i.e after 30s scrape interval as expected.
  - So Seems like the metric value is getting updated correctly as expected.

Now its time to add HPA, so we can start seeing whether scaling behaviour is working as expected

### Applying HPA Manifest
Lets Consider we are gonna scale up when RPS > 100

Apply [HPA](https://github.com/ncsham/httpserver/blob/main/hpa.yaml) Manifest via `kubectl apply -f hpa.yaml`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gohttpserver
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gohttpserver
  minReplicas: 1
  maxReplicas: 2
  metrics:
    - type: Object
      object:
        metric:
          name: http_requests_per_second
        describedObject:
          apiVersion: v1
          kind: Pod
          name: "*"
        target:
          type: Value
          value: 100000m
```

We have selected metrics type as Object, since we need to query custom metric. Below is a little description of Object metrics from K8s HPA Docs
```
These metrics describe a different object in the same namespace, instead of describing Pods. The metrics are not necessarily fetched from the object; they only describe it. Object metrics support target types of both Value and AverageValue. With Value, the target is compared directly to the returned metric from the API
``` 

Since pods/http_requests_per_second is namespaced and HPA is created in `default` namespace, HPA will query `/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second` to get the metric value. The same API which we queried initially to get the value.

### Testing Scaling Behaviour
Now its time to test whether scaling is working expected. Lets try sending 5000 requests via hey

- `watch -n 1 'kubectl describe hpa gohttpserver'`
- `date && hey -n 5000 -m GET http://localhost:20000/status && date`

#### Observations
- 2025-04-01 08:41:50 -> 5000 Requests were made 
- 2025-04-01 08:42:05 -> Time when Adapter Saw the Change
- 2025-04-01 08:42:12 -> Time When HPA saw the metric change and started scaling Up i.e 166 (5000/30) > 100 (15s default interval for HPA checking metrics)
- 2025-04-01 08:47:28 -> After 5mins, Scale Down happened due to default stabilization window of 5mins

We Can check kube-controller-manager logs as well to check when scale up or scale down happened.
```
I0401 09:45:00.062979       1 horizontal.go:881] "Successfully rescaled" logger="horizontal-pod-autoscaler-controller" HPA="default/gohttpserver" currentReplicas=1 desiredReplicas=2 reason="Pod metric http_requests_per_second above target"
```

So Basically our HPA is correctly working as Expected with Custom Prometheus Metrics.

## References
- [Adapter Github](https://github.com/kubernetes-sigs/prometheus-adapter/tree/master?tab=readme-ov-file)
- [Medium Article](https://medium.com/engineering-housing/auto-scaling-based-on-prometheus-custom-metrics-688a92e0a796)
- [End to End Walkthrough](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/walkthrough.md)
- [Configuration Walkthrough](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config-walkthrough.md)
- [HPA Autoscaling on custom metrics](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics)
- [HPA](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/)
- [Basic Golang HTTP Server](https://github.com/ncsham/httpserver)
- [Hey Load Testing Tool](https://github.com/rakyll/hey)
- [Kube Prometheus Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
