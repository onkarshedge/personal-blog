---
title: "Monitoring applications in K8s with Prometheus - 2"
date: 2020-11-13T17:58:22+05:30
draft: false
categories: ["monitoring"]
tags: ["prometheus", "grafana"]
---

#### Continued
This is part 2 of the [Monitoring applications in K8s with Prometheus]({{< ref "/prometheus-k8s" >}} "Monitoring applications in K8s with Prometheus").
In earlier post, we talked about installing prometheus in our single node kubernetes and scraped metrics of a sample Spring Boot application.
In this post, I will talk about how I build service uptime dashboard in Grafana.

#### Prometheus Metrics types
* Counter
* Gauge
* Histogram
* Summary

##### Counter
Counter is a metric whose value always increases or resets to 0 when service restarts. eg.
> http_server_requests_seconds_count{job="hello-metric",uri="/greeting"}

{{< image src="/images/prom_counter.png" caption="Http counter">}}

We are plotting the number of requests for the given URI. Prometheus scrapes at every 15 seconds. We have data-points at 15 secs apart.
The total number of requests will go on increasing, as you can see in the image. A more useful metric would be number of requests per minute.  
rate is the slope of the counter graph above at every 1m interval.  

> rate(http_server_requests_seconds_count{job="hello-metric",uri="/greeting"}[1m])

{{< image src="/images/prom_rate.png" caption="Http request rate">}}

##### Gauge
The gauge metric type can be used for values that go down as well as up, such as current memory usage. We typically won't use rate but something like an avg function.

You can read more about the other metric types in Prometheus documentation.

#### Data types
##### Range vector
What happens when we add `[1m]` in brackets ?. We get a `range_vector` in response.
Since I have given `1m` as step range, I have got 4 data-points (15 seconds apart). Think of range vector like a 2d matrix. For every time series it will have an array of values.

{{< image src="/images/prom_range.png" caption="Range vector">}}

##### Vector
Whereas when we run the query without the 1m. We get an instant vector.

{{< image src="/images/prom_vector.png" caption="vector">}}

#### Other Kubernetes metrics
Prometheus stack installs `kube-state-metrics` as one of the components. It will scrape the kube api server and get metrics about state of kubernetes objects.  
eg. kube_deployment_created, kube_job_status_completion_time, kube_pod_init_container_resource_limits_cpu_cores etc.

Let's create a Grafana dashboard to track code deployments. I could have used `kube_deployment_created` metric here,
but deployment is created once. On subsequent, helm release updates image tag is changed and a new `replica set` is created.  
Hence, there would be no data for metric `kube_deployment_created` but we would have for `kube_replicaset_created`.

> changes(max(kube_replicaset_created{replicaset=~"hello-metric.*",namespace="my-app-dev"}))

{{< image src="/images/grafana_deployment.png" caption="Deployment time">}}

* `kube_replicaset_created{replicaset=~"hello-metric.*",namespace="my-app-dev"}`: returns the timestamp when it was created. Kubernetes keeps history of 10 replica-sets by default.
 When we update the deployment, the oldest one will go away.
* `max()` will choose the latest replica set.
* `changes()` will return a vector with 1,0 when the inner max value has changed.

We don't need to specify the range here `[5m]`, grafana will take care of it. You can check the query in query inspector.  
It has added a `step` in query param. The value is chosen based on the width and pixels available for the data-points.

{{< image src="/images/grafana_query_inspector.png" caption="Query Inspector">}}


#### Services Uptime
My favourite grafana visualization is `Stat Panel`. Here is a sample services uptime dashboard.

{{< image src="/images/grafana_availability.png" caption="Service Uptime">}}

>  max(up{namespace="`${namespace}`",service=~"`${service}`"} OR on() vector(0)) * 100
>  
>  (1 - (max(up{namespace="`${namespace}`",service=~"`${service}`"} OR on() vector(0)))) * ($__range_s / 60)


The first query gives the uptime percentage. The second query gives the downtime.  
In both the queries, I have `OR on() vector(0)` because if there is no data for some time in between it would still show as 100.  
`$__range_s` is the time in seconds of the range interval. In field options, I have given mean instead of last, min, max to get accurate result.












