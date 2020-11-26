---
title: "Monitoring applications in K8s with Prometheus - 3"
date: 2020-11-15T16:20:12+05:30
draft: false
categories: ["prometheus", "grafana"]
tags: ["monitoring"]
---

### Continued
This is part 3 of the [Monitoring applications in K8s with Prometheus]({{< ref "/prometheus-k8s" >}} "Monitoring applications in K8s with Prometheus").
In part-2, we got to know different types of prometheus metrics and created a service uptime visualization in Grafana.  
In this post, we will see what is `kube-state-metrics` and create a Deployment time chart in Grafana.

### Kubernetes Objects State
Prometheus helm chart installs `kube-state-metrics` as one of the components. It will scrape the kube api server and get metrics about state of kubernetes objects.
Like kubectl in watch state but for all objects.    
eg. kube_deployment_created, kube_job_status_completion_time, kube_pod_init_container_resource_limits_cpu_cores etc.  
More information available [here](https://github.com/kubernetes/kube-state-metrics/tree/master/docs)[1]

### Tracking Code Releases
#### Selecting a metric
CI/CD pipeline would be the best place to check code releases. Here we want to use one of the kube-state-metrics and create a time chart visualization.  
We can use `kube_deployment_created` metric here, but a deployment is created once.
On subsequent, helm release updates, image tag is changed and a new `replica set` is created.  

The value returned by metric `kube_deployment_created` would be same throughout. Hence, we will use `kube_replicaset_created`.
kube_replicaset_created returns the UTC timestamp of the replicaset creation.

> kube_replicaset_created{replicaset=~"hello-metric.*",namespace="my-app-dev"}

We will see the creation timestamp of each replicaset. Kubernetes keeps history of 10 replicasets by default.
Hence, we may see upto 10 values at a given instant.

{{< image src="/images/prom_max_replicaset.png" caption="Replicaset created">}}

We see 4 replicasets created at different times. The Y axis is formatted in `G` because of large value.
I used max by replicaset just to select the label replicaset in final output.
The value remains same throughout. We are interested in the time it first appeared.  

#### Converting lines to single points
Consider only a single replicaset for simplicity. The query returns values in following format. [X-axis timestamp, Y-axis value]. 
> Array[1603166400, 1603166380]  
> Array[1603166430, 1603166380]  
> Array[1603166460, 1603166380]  
> Array[1603166490, 1603166380]  
> Array[1603166520, 1603166380]  

The scraping interval is of 30 seconds. Each X-axis timestamp value is 30 seconds apart. The Y-axis value is the timestamp of replicaset created.  
We are interested in the first point `Array[1603166400, 1603166380]`. The difference between the X and Y value will be at most the scrape interval.
The difference here is `20`. For the subsequent points the difference goes on increasing.  

We can only compare metric values in PromQL. To bring the timestamp in Y-axis field, we can use `timestamp()` function.
> timestamp(max(kube_replicaset_created{replicaset=~"hello-metric.*",namespace="my-app-dev"}) by (replicaset))

> Array[1603166400, 1603166400]  
> Array[1603166430, 1603166430]  
> Array[1603166460, 1603166460]  
> Array[1603166490, 1603166490]  
> Array[1603166520, 1603166520]  

Now we can subtract the two vectors, and the difference will be less than equal to scrape interval for the first point.

{{< image src="/images/prom_max_replica_set_shift.png" caption="Replicaset created point">}}

If we watch closely, we will see 4 points. Zoom in, if not visible. I have given 120 so that the points are visible.
{{< admonition warning >}}  
However, the Y-axis values are still timestamps.  
{{</ admonition >}}

#### Visualizing in Grafana
Let's take this query to grafana.

{{< image src="/images/grafana_replicaset_lines.png" caption="Replicaset created grafana">}}

2 changes in the query here:
1) **bool** : In prometheus the Y-axis value was still the metric value(timestamp). `bool` will select all the values from the left
and convert to 0, if false 1, if true.
2) **__interval_ms**: Grafana will dynamically choose a step/interval based on the number of pixels available to plot.
Hence, we need a dynamic value for comparison. We cannot hardcode 60 or 120.

> Array[1603080000, 1602850335]  
  Array[1603081800, 1602850335]  
  Array[1603083600, 1603083574]  

Above, the X-axis values are 30 min apart. Hence, we need (1800 * 2) = 3600 value in the query.  

We can see in the below image, the step is 15s, and the __interval_ms value in the query is replaced with 15000.

{{< image src="/images/grafana_replicaset_query_inspector.png" caption="Query Inspector">}}

#### Final PromQL Query in Grafana

{{< image src="/images/grafana_replicaset_promql.png" caption="Code deployments">}}
 
### VictoriaMetrics MetricsQL
There is an alternate approach : **MetricsQL**. In my project we were using [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics)[2].
[MetricsQL](https://github.com/VictoriaMetrics/VictoriaMetrics/wiki/MetricsQL)[3] is an extension to PromQL. It is backward compatible with PromQL. 

#### Step 1
`kube_replicaset_created{replicaset=~"hello-metric.*",namespace="my-app-dev"}` over a longer period would like this.
So many replicasets!!.

{{< image src="/images/grafana_replicaset_dev.png" caption="Replicaset created longer range">}}

#### Step 2
`max(kube_replicaset_created{replicaset=~"hello-metric.*",namespace="my-app-dev"})` would get the max value at each point.
Whenever the max value has changed a new replicaset was created.
{{< image src="/images/grafana_replicaset_max.png" caption="Replicaset Max">}}

{{< admonition type=info title="changes" >}} 
There is a `changes()` function available in PromQL 2.2, but it takes a range vector as an input.
It returns a vector with 1,0 when the inner max value has changed.
`max()` returns an instant vector. We cannot put [5m] in front of max function. Prometheus will return error invalid query.
{{</ admonition >}}

#### Step 3
`changes(max(kube_replicaset_created{replicaset=~"hello-metric.*",namespace="my-app-dev"})[5m])`

This is an invalid query in `PromQl`. However, it is a valid MetricsQL.    
I spent a day debugging why is this working in Grafana. Later got to know from Infra team, the data source is VictoriaOps and not Prometheus.

{{< image src="/images/grafana_deployment.png" caption="Deployment time">}}

{{< image src="/images/grafana_deployment_victorops.png" caption="Deployment time with query view">}}

Using the `$__interval_` instead of hardcoded `[5m]`. If the step happens to be 30min, `changes` function would not find any value change is span of 5min.
Also, it is good to specify min step. If the step happens to be 15s, the graph would not look good.[4]

### Links
1) https://github.com/kubernetes/kube-state-metrics/tree/master/docs
2) https://github.com/VictoriaMetrics/VictoriaMetrics
3) https://github.com/VictoriaMetrics/VictoriaMetrics/wiki/MetricsQL
4) https://www.youtube.com/watch?v=09bR9kJczKM