---
title: "Monitoring applications in K8s with Prometheus - 2"
date: 2020-11-13T17:58:22+05:30
draft: false
categories: ["prometheus", "grafana"]
tags: ["monitoring"]
---

### Continued
This is part 2 of the [Monitoring applications in K8s with Prometheus]({{< ref "/prometheus-k8s" >}} "Monitoring applications in K8s with Prometheus").
In earlier post, we talked about installing prometheus in our single node kubernetes and scraped metrics of a sample Spring Boot application.  
In this post, we will see how to build a service uptime dashboard in Grafana.

### Querying
PromQL expression consists of metric_names filtered by labels, operators, functions.  
1) `metric_name{label1="label_value"}` will return multiple time series matching those label values. We can filter these timeseries by giving more specific labels.
We can use `!=` (not equal to), `=~` (for regex matching).
2) operators are binary operators (+, -, *, /, %, ^), logical operators, aggregation operators (min, max, avg, sum etc.)
3) functions: some functions take input vectors as parameters, some take range vectors. eg. ceil(), rate(), increase() etc.

We will see them in the examples.

### Prometheus Data types
#### Instant Vector
Instant vector - a set of time series containing a single sample for each time series, all sharing the same timestamp.
When we type `http_server_requests_seconds_count{job="hello-metric",uri="/greeting"}` we get an instant vector.

{{< image src="/images/prom_vector.png" caption="vector">}}

If we inspect in browser tools, we will see that the current timestamp is sent as a query parameter. In response, we get, `resultType: vector`

#### Range vector

> http_server_requests_seconds_count{job="hello-metric",uri="/greeting"}[1m]

1m is the range of 1 minute.  
What happens when we add `[1m]` in brackets ?. We get a `range_vector` in response.
Since we have scrape interval of 15 seconds, we will get 4 data-points (15 seconds apart). Think of range vector like a 2d matrix. For every time series it will have an array of values.

{{< image src="/images/prom_range.png" caption="Range vector">}}

#### Scalar
Scalar - a simple numeric floating point value. Prometheus exports every metric value as float. 

### Prometheus Metrics types
* Counter
* Gauge
* Histogram
* Summary

When we visit the metrics endpoint. We may see a couple of comments on top of the metric.  
`# HELP`: Description and `# Type`: One of (Counter, Gauge, Histogram, Summary) 

#### Counter
Counter is a metric whose value always increases and resets to 0 when service restarts.
Here is a log4j2 events counter.  
```Prometheus Metric
# HELP log4j2_events_total Number of fatal level log events
# TYPE log4j2_events_total counter
log4j2_events_total{level="warn",} 0.0
log4j2_events_total{level="debug",} 20.0
log4j2_events_total{level="error",} 3.0
log4j2_events_total{level="trace",} 0.0
log4j2_events_total{level="fatal",} 0.0
log4j2_events_total{level="info",} 8.0
``` 

eg of another metric counter: Number of http requests.
> http_server_requests_seconds_count{job="hello-metric",uri="/greeting"}

{{< image src="/images/prom_counter.png" caption="Http counter">}}

When we switch to graph mode, and choose a range of suppose last 12h, we see the following graph.
We are plotting the number of requests for the given URI. Prometheus has scrapped metric at 15 secs apart, but if we choose a large
range eg. days It will not be possible to show every data point. Hence, the UI will send an API request with a suitable interval.  
`/api/v1/query_range?query=<some_query_here>&start=1748140123&end=179252511&step=600`  
The step in the above example is 10 minutes or 600 seconds. We will get data points 10 min apart.  
 
The total number of requests will go on increasing, as we can see in the image. A more useful metric would be change in number of requests per second.    

`rate` is the slope of the counter graph above between two data points 1m apart, calculated at every interval.

> rate(http_server_requests_seconds_count{job="hello-metric",uri="/greeting"}[1m])

{{< image src="/images/prom_rate.png" caption="Http request rate">}}

{{< admonition type=question title="Why did we choose 1 minute ?. What number should we use ?" open=true >}}
To calculate slope we need at least 2 points. Using a range window of 2 times the scrape interval i.e 30 seconds is risky.
If we are lucky we may find 3 points each matching exactly the scrape timestamp, as the range is inclusive.
But most likely we will find 2 points.  
Prometheus will try very hard to scrape exactly at the scrape interval. But due to network delays, there may be slight delay in the scrapes.
Or due to failures, a scrape might be missed, and we may end up with only one point.  
**Hence chosen a 4 x scrape_interval = 1m**
{{< /admonition >}}

Below is a code snippet of `rate function` implementation. There is an `if` check for at least 2 points. Also, I have skipped the additional code of extrapolation.
``` go {linenos=table,hl_lines=[13,21],linenostart=59}
func extrapolatedRate(vals []parser.Value, args parser.Expressions, enh *EvalNodeHelper, isCounter bool, isRate bool) Vector {
	ms := args[0].(*parser.MatrixSelector)
	vs := ms.VectorSelector.(*parser.VectorSelector)

	var (
		samples    = vals[0].(Matrix)[0]
		rangeStart = enh.Ts - durationMilliseconds(ms.Range+vs.Offset)
		rangeEnd   = enh.Ts - durationMilliseconds(vs.Offset)
	)

	// No sense in trying to compute a rate without at least two points. Drop
	// this Vector element.
	if len(samples.Points) < 2 {
		return enh.Out
	}
	// ....
	resultValue := lastValue - samples.Points[0].V + counterCorrection
    // ....
	resultValue = resultValue * (extrapolateToInterval / sampledInterval)
	if isRate {
		resultValue = resultValue / ms.Range.Seconds()
	}
	return append(enh.Out, Sample{
		Point: Point{V: resultValue},
	})
}
```  


#### Gauge
The gauge metric type can be used for values that go down as well as up, such as current memory usage, number of items in a queue.
We usually won't use rate with gauge but something like an `avg_over_time` function.

```Prometheus Metric
# HELP system_cpu_count The number of processors available to the Java virtual machine
# TYPE system_cpu_count gauge
system_cpu_count 8.0
# HELP http_server_requests_seconds_max  
# TYPE http_server_requests_seconds_max gauge
http_server_requests_seconds_max{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/metrics",} 0.003985642
http_server_requests_seconds_max{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",} 8.008833202
```
> http_server_requests_seconds_max: The maximum time for a request in seconds. Above we can see it is 8 for `/greeting`
> because I have put a `Thread.sleep(randomValue)`;

You can find the code at [hello-metric](https://github.com/onkarshedge/hello-metric)

#### Histogram
A histogram will sample the observations and put them into configured buckets. Used for request time, response sizes.  
In the application.properties is the below configuration:  
```properties
management.metrics.distribution.percentiles-histogram.http.server.requests=false
management.metrics.distribution.slo.http.server.requests=200ms,500ms,750ms,1000ms,1500ms,2000ms
```
The default histogram buckets configuration is very granular. I have overridden it.

```Prometheus Metric
# HELP http_server_requests_seconds  
# TYPE http_server_requests_seconds histogram
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",le="0.2",} 1.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",le="0.5",} 3.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",le="0.75",} 5.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",le="1.0",} 5.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",le="1.5",} 7.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",le="2.0",} 8.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",le="+Inf",} 16.0
http_server_requests_seconds_count{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",} 16.0
http_server_requests_seconds_sum{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",} 33.30292711
```
We can see 3 metrics:
1) http_server_requests_seconds_bucket{le=<upper_bound>}
2) http_server_requests_seconds_count{} : Total number of requests (counter)
3) http_server_requests_seconds_sum{} : Sum of request durations (counter)

Histogram bucket counter are cumulative. Number of requests with less than 200ms duration is 1. Number of requests with less 500ms (including the le=0.2 bucket) is 3.
It is **not** between 200ms and 500ms.  

We can see the timeseries for each bucket(label le) 0.2, 0.5, 0.75, +Inf etc. Since it is cumulative the +Inf will be on top.
{{< image src="/images/prom_histogram_buckets.png" caption="Histogram buckets count">}}

We can find the median(0.5), 3rd quantile, 95th percentile of the response times.
{{< image src="/images/prom_histogram_quantile.png" caption="Histogram Quantile">}}

Histogram bar chart, showing the distribution of response times. Note the format is *Heatmap*
{{< image src="/images/grafana_histogram_bar.png" caption="Histogram Bar Chart">}}

With Heatmap visualization, we can view the change in histogram over time. In data format, choose timeseries buckets[3].
On Y-axis we have our buckets and X-axis interval of 1min. Each vertical slice is one histogram. Since it is a counter, we will have to use increase.
{{< image src="/images/grafana_histogram_heatmap.png" caption="Histogram Heatmap">}}

> rate(http_server_requests_seconds_sum{outcome="SUCCESS",uri="/greeting"}[1m])   
> **/**  
> rate(http_server_requests_seconds_count{outcome="SUCCESS",uri="/greeting"}[1m]) 

This will give the average http request duration for greeting endpoint. Using rate as both are counters.

#### Summary
Summary and Histogram both are used to calculate quantiles. Summary was used before Histogram.
Both are similar with differences:
* Percentiles are calculated on the client side whereas in Histogram they are computed on Prometheus server.
* Histogram requires you to preselect buckets, summary does not.

More details on difference [here](https://prometheus.io/docs/practices/histograms/)[1]    
You can read more about the metric types in Prometheus [documentation](https://prometheus.io/docs/concepts/metric_types/)[2].


### Services Uptime
Let's create a Services Uptime dashboard in Grafana.


{{< admonition type=question title="How do we know if a service is up or down ?" open=true >}}
We can use the `up` metric.  
`up{job="hello-metric"}` will return multiple time series, one for each pod. The value will be 1 or 0.  
1 indicates healthy (prometheus was able to scrape successfully).  
0 indicates unhealthy
{{< /admonition >}}

Clarification of some labels before moving ahead.
* Job: Each target scraped will have a job label. eg. `job=hello-metric`
* Instance: A job can have multiple replicas. In our context, in k8s there are multiple pods for a deployment.
 The instance label value is `<host_ip>:<port>` eg. `instance=10.1.0.16:8080`

> Metrics coming from both the pods will have same job label values but different instance label values.    
> A service is up if any of the pods is reachable. If one pod is up and other is down. max() will return 1.    
> Hence, using max() function.

My favourite Grafana visualization is `Stat Panel`. We can write multiple queries. The stat panel can be repeated for each service value.
{{< image src="/images/grafana_availability.png" caption="Service Uptime">}}

##### Availability percentage
`max(up{namespace="${namespace}",service=~"${service}"}) * 100`

> 114:Array[1603080000,100]  
  115:Array[1603081800,100]  
  116:Array[1603083600,100]  
  117:Array[1603085400,100]  
  118:Array[1603087200,0]  
  119:Array[1603089000,0]  
  120:Array[1603090800,0]  
  121:Array[1603092600,0]  
  122:Array[1603094400,0]  
  123:Array[1603096200,100]  
  124:Array[1603098000,100]  
  125:Array[1603099800,100]  
  126:Array[1603101600,100]  
  127:Array[1603103400,100]  

Multiplying each {1, 0} value with 100 will give an array of values with {100, 0}.  
In Fields -> set unit to percentage,  value -> mean.  
As we need a single value in the stat panel, we need to specify an aggregation function. Mean is correct among (last value, sum, min, max etc.)

##### Downtime
`(1 - (max(up{namespace="${namespace}",service=~"${service}"}))) * ($__range_s / 60)`

> **__range_s** value is the time range in seconds, choosen in the global time filter. eg. If we have set the time filter as last 30 days,
> 30 days = 25,92,000 seconds. Dividing by 60 to get minutes. As we don't need small resolution of downtime in seconds.  
> This is a grafana feature, it will substitute the value in the query before sending the http request. 
> We can verify this in the query inspector (Available in Grafana 7).

In field options override change the unit to time minutes. We need to specify in field overrides because the units are different for both.  
Threshold colors: The stat panel looks beautiful because of the colors. We can assign colors based on a threshold range.  
0 - 50 -> red  
50 - 75 -> orange  
75 - 90 -> yellow  
90 - 100 -> green  

##### Final queries

>  max(up{namespace="`${namespace}`",service=~"`${service}`"} OR on() vector(0)) * 100
>  
>  (1 - (max(up{namespace="`${namespace}`",service=~"`${service}`"} OR on() vector(0)))) * ($__range_s / 60)
 
If there is no data available for this metric it would show as Not Data available.
Or if the service was deleted for some time in between that would not get counted as downtime.  
In both the queries, I have `OR on() vector(0)` to treat no data as 0.

### Next
We will use kube-state-metrics to get information about change of Kubernetes objects.  
[Part-3]({{< ref "prometheus-k8s-three" >}} "Part-3")

### Links
1) https://prometheus.io/docs/practices/histograms/
2) https://prometheus.io/docs/concepts/metric_types/
3) https://grafana.com/blog/2020/06/23/how-to-visualize-prometheus-histograms-in-grafana/

<!--

# HELP http_server_requests_seconds  
# TYPE http_server_requests_seconds summary
http_server_requests_seconds_count{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/metrics",} 6.0
http_server_requests_seconds_sum{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/metrics",} 0.038895665
http_server_requests_seconds_count{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",} 10.0
http_server_requests_seconds_sum{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/greeting",} 61.117478925

-->






