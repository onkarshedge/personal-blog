---
title: "Monitoring applications in K8s with Prometheus"
date: 2020-11-13T14:33:38+05:30
draft: false
categories: ["monitoring"]
tags: ["prometheus", "grafana"]
---

#### Versions used
> Docker for desktop: 2.3.0  
> Kubernetes: 1.16.5  
> Prometheus: helm-chart v9.0  
> Spring boot: 2.3.4  
> Java : 11  

#### Sample Application with metrics in Prometheus format
You can have a sample application in any language which emits metrics in Prometheus format.
I have a hello-world spring boot application. We will have to add prometheus dependency in build.gradle.
```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'org.springframework.boot:spring-boot-starter-web'

	implementation 'io.micrometer:micrometer-registry-prometheus'
}
```
And the following in application.properties.
``` properties
management.endpoints.web.exposure.include=info, health, prometheus
management.endpoints.web.path-mapping.prometheus=metrics
management.metrics.enable.all=true
```
When you visit `/actuator/metrics` path you will see metrics in prometheus format. Something like below. Prometheus metric format has become popular that 
is also used in OpenMetrics project. 
> The format is: `<metric name>{<label name>=<label value>, ...}`
``` prometheus
# TYPE http_server_requests_seconds summary
  http_server_requests_seconds_count{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/metrics",} 1.0
  http_server_requests_seconds_sum{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/metrics",} 0.045635932
  http_server_requests_seconds_count{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",} 45.0
  http_server_requests_seconds_sum{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",} 0.063271443
  jvm_memory_max_bytes{area="nonheap",id="Metaspace",} -1.0
  jvm_memory_max_bytes{area="nonheap",id="CodeHeap 'non-nmethods'",} 7553024.0
  jvm_memory_max_bytes{area="heap",id="G1 Eden Space",} -1.0
  jvm_memory_max_bytes{area="nonheap",id="Compressed Class Space",} 1.073741824E9
```

#### Installing Prometheus
I will be installing prometheus and it's components in `monitoring` namespace and my hello-world application in `my-app-dev`.
```shell script
kubectl create ns monitoring
kubectl create ns my-app-dev
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
kubectl get all -n monitoring
```

You will see a lot of components installed: 
 
| No. | k8s object | description |
| --- | ---------- | ----------- |
| 1 | daemonset.apps/prometheus-prometheus-node-exporter | Prometheus exporter for hardware and OS metrics exposed by *NIX kernels |  
| 2 | deployment.apps/prometheus-grafana | This has some ready-made dashboards for Pods, Namespace(workload) monitoring etc. | 
| 3 | deployment.apps/prometheus-kube-prometheus-operator | The Prometheus Operator uses Kubernetes custom resources to simplify the deployment and configuration of Prometheus, Alertmanager, and related monitoring components. | 
| 4 | deployment.apps/prometheus-kube-state-metrics | kube-state-metrics is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects. Like kubectl in watch state but for all objects. | 
| 5 | statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus |  Prometheus  | 
| 6 | statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager | AlertManager to configure alerts based on rules. |

Let's access the Prometheus UI and Grafana UI. Visit [http://localhost:9090](http://localhost:9090), [http://locahost:3100](http://locahost:3100) respectively.
The credentials for grafana are `username: admin` & `password: prom-operator`. You can also get the password from k8s secrets.

```shell script
kubectl port-forward -n monitoring pod/prometheus-prometheus-kube-prometheus-prometheus-0  9090
kubectl port-forward -n monitoring service/prometheus-grafana 3100:80
```

Browse the Prometheus UI. In Status/Targets, You shall find some predefined Targets. Under Configuration, you will find:  
``` 
global:
  scrape_interval: 30s
  scrape_timeout: 10s
  evaluation_interval: 30s
```

In Grafana, you shall find some dashboards already configured for Nodes, Namespace workloads, Pods, Kube API server etc. Here is a CPU usage sample.

{{< image src="/images/prom_grafana_sample.png" caption="CPU Usage">}}

The query looks complicated and doesn't match to any of the metrics (neither application/kube-stat-metrics/node etc...).
We will talk about it in [later section](#appendix).

#### Deploying our application 

Build image with `./gradlew bootBuildImage`. Spring boot 2.3 has this gradle task which creates optimised docker image.
Deploy in `my-app-dev` namespace that we created earlier.  
**Deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "hello-metric"
  namespace: my-app-dev
  labels:
    app: "hello-metric"
spec:
  replicas: 2
  template:
    metadata:
      name: "hello-metric"
      labels:
        app: "hello-metric"
    spec:
      containers:
        - name: "hello-metric"
          image: "hello-metric:1.0"
          imagePullPolicy: Never
          ports:
            - containerPort: 11000
              name: hello-port
              protocol: TCP

      restartPolicy: Always
  selector:
    matchLabels:
      app: "hello-metric"

```

**Service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-metric
  labels:
    app: hello-metric
  namespace: my-app-dev
spec:
  selector:
    app: hello-metric
  ports:
    - port: 11000
      targetPort: hello-port
      name: hello-svc-port
  type: ClusterIP
```

The application is deployed in `my-app-dev` namespace.
It won't be accessible on host machine. You can either port-forward or make it NodePort Service to access on host machine.

#### Collecting metrics of our application
{{< admonition type=question title="Question" open=true >}}
Coming to the main part how to tell prometheus to scrape metrics at `/actuator/metrics` endpoint of our applicaiton ?.
{{</admonition>}}
Answer: *Service Monitor*  
The helm install of prometheus stack created some CRDs. One of which is service monitor.
Service monitor is where we declare from which service to scrape metrics and the interval. Like other k8s resources it also works on matching labels.
I am creating this service monitor in monitoring namespace.  
One important thing is the prometheus object in monitoring namespace wouldn't know about this service monitor until you have the label 
`release: prometheus`. If you describe prometheus resource. `kubectl describe prometheus -n monitoring` you will find that it matches service monitors on this label.
```yaml
  Service Monitor Selector:
      Match Labels:
        Release:  prometheus
```  
Create the following Service monitor in monitoring namespace.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: monitoring-hello
  namespace: monitoring
  labels:
    app: hello-metric
    release: prometheus # imp had forgot this, if you do describe prometheus resource there is a servicemonitor selector for this label
spec:
  selector:
    matchLabels:
      # Target app service
      app: hello-metric
  endpoints:
    - interval: 15s
      path: '/actuator/metrics'
      port: hello-svc-port
      scheme: http
  namespaceSelector:
    matchNames:
      - my-app-dev
```
Wuhoo!!, In targets, we should see our target hello-metric. 
{{< image src="/images/prom_target_hello.png" caption="Hello Target">}}

In promql you can execute a sample query: `jvm_memory_used_bytes{job="hello-metric"}`.    
You can filter metrics with labels. The output is shown only for that instant. In range tab you can give a time range.
We will talk more about the PromQL in [Part-2]({{< ref "prometheus-k8s-two" >}} "Part-2"), and I will show a sample service availability dashboard in Grafana.

#### Appendix{#appendix}
The PromQL Query of the Grafana CPU usage dashboard uses custom metric expression. Under rules in prometheus you shall find `alerting.rules` and `recording.rules`.  
From Prometheus documentation:  
> Recording rules allow you to precompute frequently needed or computationally expensive expressions and save their result as a new set of time series.
> Querying the precomputed result will then often be much faster than executing the original expression every time it is needed.
> This is especially useful for dashboards, which need to query the same expression repeatedly every time they refresh.
