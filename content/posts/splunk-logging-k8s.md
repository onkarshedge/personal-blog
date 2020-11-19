---
title: "Logging in K8s with Splunk"
date: 2020-11-16T11:44:46+05:30
draft: true
---

#### Why Splunk ?
This post is not a comparative study of logging tools eg. ELK vs Splunk vs SumoLogic. I haven't used other logging tools.
In this post, We will:
1) Track logs from a file, create a visualization.
2) Forward logs of an application deployed in Kubernetes to Splunk.
3) Monitor K8s objects and events
4) Gather Metrics from K8s

{{< admonition type=info title="" open=true >}}
Splunk: 8.0.3  
Kubernetes: v1.16.5
splunk-connect-for-kubernetes (helm-chart): 1.4.3 
Docker for desktop: 2.3.0.2  
Docker Engine: 19.03.8  
OS: Catalina 10.15.4  
{{< /admonition >}}

#### Run Splunk
Run Standalone Splunk in a docker container.  
``` shell
docker run --name splunk -d -v `pwd`/splunk_data:/splunk_data -v `pwd`/fields.conf:/opt/splunk/etc/system/local/fields.conf -p 8000:8000 -p 8089:8089 -p 8088:8088 -e 'SPLUNK_START_ARGS=--accept-license' -e 'SPLUNK_PASSWORD=splunk_admin' splunk/splunk:latest
```
Options:
* In the local directory `splunk_data` there is a `earthquakes.csv` file. Downloaded from Splunk examples [here](https://docs.splunk.com/DocumentationStatic/WebFramework/1.0/earthquakes.csv). We are mounting so that we can add it as a file source in Splunk.
*  `fields.conf` is a configuration file. The contents of this file are explained [here](#fields-conf).
* What are these 3 ports exposed ?: 
  * 8000 is the Splunk UI.
  * 8088 is the Http Event Collector(HEC) port.
  * 8089 is SplunkD management port (REST API access).  

If you are just interested in K8s logs, Remove the splunk_data volume mount and skip to section [Forward K8s logs to Splunk](#k8s-logs)
```shell 
docker run --name splunk -d -v `pwd`/fields.conf:/opt/splunk/etc/system/local/fields.conf -p 8000:8000 -p 8089:8089 -p 8088:8088 -e 'SPLUNK_START_ARGS=--accept-license' -e 'SPLUNK_PASSWORD=splunk_admin' splunk/splunk:latest
```

Visit [http://localhost:8000](http://localhost:8000). Log in with **username**: `admin`, **password**: `splunk_admin`
For more information please refer to docker-splunk [documentation page](https://splunk.github.io/docker-splunk/)[1].

We will configure two sources File, Kubernetes.  
#### Nomenclature
* **Source type**: Type of data, so that Splunk can format intelligently. Helps in searching eg. nested fields in XML/JSON can be searched with `spath` command.
Some examples of sourcetype are csv, tsc, psv, json, statsd, dmseg etc.
* **App context**: Folders within Splunk for manageability and separation of domain, configuration.
* **Index**: Reside in flat files on the Splunk platform instance known as the indexer. Splunk transforms the data into searchable events and stores those events in the index.
Consider creating multiple indexes based on different access privileges, retention periods, or logical groupings.
#### File/Directory as a datasource
We will configure the `splunk_data/earthquakes.csv` as data source. These are the first few lines of the file.
``` csv
 Src,Eqid,Version,Datetime,Lat,Lon,Magnitude,Depth,NST,Region
  ci,15281913,0,"Wednesday, February  6, 2013 00:11:58 UTC",33.2985,-116.3707,1.4,9.60,82,"Southern California"
  ci,15281897,0,"Tuesday, February  5, 2013 23:50:14 UTC",33.2987,-116.3720,1.5,9.60,70,"Southern California"
  nc,71932170,0,"Tuesday, February  5, 2013 23:37:54 UTC",37.5620,-122.4353,1.1,10.30, 9,"San Francisco Bay area, California"
```

Goto Settings -> Data Inputs -> Local Inputs -> File & Directories -> New File & Directory. Browse the earthquakes.csv file. Splunk can continuously monitor the directory or index once.
We can also monitor remote files with forwarding & receiving configuration.
{{< image src="/images/splunk_file_input.png" caption="Splunk File Data Input">}}

In `Set Source Type` screen, do the following:  
* Set sourcetype CSV
* Timestamp fields: Datetime
* Timestamp format: %A, %B %d, %Y %H:%M:%S %Z 
* Under Advanced setting add new setting MAX_DAYS_AGO: 10000 since the data is from 2013.
* Save the sourcetype as `earthquake:csv`, if you specify csv it will override the existing csv in system app.
* Save in whichever app you want.
* Specify a new index `my_app`. Set the index data type as `events`.

Since the Datetime column values are in a specific format eg. "Tuesday, February  5, 2013 23:50:14 UTC", We have to tell Splunk the time column and the format.
More on [timeformat](https://docs.splunk.com/Documentation/SplunkCloud/8.1.2008/SearchReference/Commontimeformatvariables)[2]

After you save, go to Search and type `index=my_app sourcetype="earthquake:csv"`
{{<image src="/images/splunk_sample_search.png" caption="Splunk sample search">}}

The time filter is `All time` that should be okay as there are only 981 events. However, in a real scenario we would use a time filter of days/months based on the number of events.
  
On the left, under Interesting fields, we see some additional fields like day_month, date_wday etc. Splunk will add these [default fields](https://docs.splunk.com/Documentation/Splunk/8.1.0/Knowledge/Usedefaultfields)[3] for us.
 So that we would not have to do time formatting if required.
If we click on any field, we will see top interesting values for that field.

Let's filter events between this region. `index="my_app" sourcetype="earthquake:csv" | search Lat>=22.50000 Lat<45.00000 Lon>=45.00000 Lon<90.00000`
We can also view a statistics table with following query `index="my_app" sourcetype="earthquake:csv" Depth>9 | stats count by Region`  
Or a map visualization with `index="my_app" sourcetype="earthquake:csv" | geostats latfield=Lat longfield=Lon count`

{{<image src="/images/splunk_earthquake_map.png" caption="Earthquake Map">}}
I will show more visualizations in the next post.

#### Forward K8s logs to Splunk{#k8s-logs}

##### HEC token
The HEC(Http Event Collector token) will be used by the SCK(Splunk Connect for Kubernetes) to forward data to Splunk server over http.
* Create index `my_app` (type: events) if not created earlier.
* Create a new index for metrics type. (Splunk -> Settings -> Indexes). `my_app_metrics`.
* Go to Settings -> Data Inputs -> HEC. Create a new HEC token and specify the indices `my_app`, `my_app_metrics`.
* In Data Inputs -> HTTP Event Collector we will see our token. 
    
    
##### Installing SCK
Splunk Connect for Kubernetes provides a way to import and search Kubernetes application logs, objects, and metrics data in Splunk.
  Here are the contents of sck_values.yaml. **Replace the token**. Cluster name can be anything. 
```yaml
global:
  splunk:
    hec:
      protocol: https
      insecureSSL: true
      token: a78b4e7d-9deb-4130-bfa2-f81b3b144ccb
      host: host.docker.internal
      port: 8088
      indexName: my_app
  kubernetes:
    clusterName: "k8sdev"

splunk-kubernetes-logging:
  journalLogPath: /run/log/journal
  splunk:
    hec:
      indexName: my_app

splunk-kubernetes-objects:
  objects:
    core:
      v1:
        - name: pods
        - name: namespaces
        - name: nodes
        - name: services
        - name: config_maps
        - name: persistent_volumes
        - name: service_accounts
        - name: persistent_volume_claims
        - name: resource_quotas
        - name: component_statuses
        - name: events
          mode: watch
    apps:
      v1:
        - name: deployments
        - name: daemon_sets
        - name: replica_sets
        - name: stateful_sets
  splunk:
    hec:
      indexName: my_app

splunk-kubernetes-metrics:
  kubernetes:
    insecureSSL: true
  splunk:
    hec:
      indexName: my_app_metrics
```

More detailed values.yaml can be found [here](https://github.com/splunk/splunk-connect-for-kubernetes/blob/main/helm-chart/splunk-connect-for-kubernetes/values.yaml)[4]
 
{{< admonition type=note  open=true >}}
The host value is `host.docker.internal`. Since the Splunk is running outside k8s it would not be discoverable by K8s DNS service.
However, both are running on docker-for-desktop, We can use host.docker.internal.
{{</admonition>}}

Create two namespaces: logging, my-app-dev. We will install SCK in logging namespace and in my-app-dev a sample application.

```shell script
kubectl create ns my-app-dev
kubectl create ns logging
helm repo add splunk https://splunk.github.io/splunk-connect-for-kubernetes/
helm install splunk --namespace=logging -f sck_values.yaml splunk/splunk-connect-for-kubernetes
```
`kubectl get all -n logging`. We shall see the following objects created.

|Object| Description|
| ---- | ---------- |
| daemonset.apps/splunk-splunk-kubernetes-logging | Splunk uses node logging agent method. The daemonset leverages fluentd to get the logs of containers, transforms them in Splunk friendly format with `jq` and forwards it to Splunk via HEC.  |
| deployment.apps/splunk-splunk-kubernetes-objects | Splunk also collects the K8s object data. The spec and the status of objects like Deployemnt, Job, Pod, Service etc. The entire list is in the `sck_values.yaml` |
| daemonset.apps/splunk-splunk-kubernetes-metrics | Metrics Server collects resource metrics from kubelets and exposes them in Kubernetes apiserver through Metrics API metrics.k8s.io/v1beta1. We can use `kubelet top` command to see the node/cpu metrics.  
|| [Fluentd metrics plugin](https://github.com/splunk/fluent-plugin-kubernetes-metrics) collects the metrics, formats the metrics for Splunk ingestion by assuring the metrics have proper metric_name, dimensions, etc., and then sends the metrics to Splunk using out_splunk_hec Fluentd engine |

#### Logging Architecture

Application running in a container when logs to `stdout` & `stderr` is handled by the container runtime. In case of Docker container engine, these two streams are handled by the logging-driver.  
Here is a list of supported [logging drivers](https://docs.docker.com/config/containers/logging/configure/)[5].  
> Run `docker info` on the node, and you will find the default logging driver as `Logging Driver: json-file`.

`json-file` logging driver configuration will write logs from a container in json-format in a file at path
`/var/lib/docker/containers/<container-id>-json.log` on the node/host machine.  
In `/var/log/pods/<namespace_name>_<pod_name>_<pod_id>/<pod_name>/0.log` file we will find the logs of the pod. This file is a symbolic link.
`pods/my-app-dev_hello-metric-599c644944-xdvtg_0a82a499-567b-42d1-a0e1-624c254ddc4d/hello-metric/0.log -> /var/lib/docker/containers/<container_id>-json.log`

Here is sample log event from the file.
``` json
{
"log":"2020-11-17 13:09:06.880  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 11000 (http) with context path ''\n",
"stream":"stdout",
"time":"2020-11-17T13:09:06.8819219Z"
}
```
When we log from the application in JSON format the `log` field's value will be a JSON string.
> splunk-kubernetes-daemonset will use fluentd agent to monitor the log files in /var/log/pods/ , transform using jq and send to Splunk server with the HEC token.


{{< admonition type=info open=true >}}
minikube ssh equivalent for docker desktop is `docker run -it --privileged --pid=host justincormack/nsenter1`
{{< /admonition >}}

#### Application
Now let's get the logs of applications running in kubernetes on Splunk. We need an application which logs to stdout in JSON format. It can be any other format(key-value pair, XML etc). A structured format provides readability and is useful in querying.
I have a sample Spring boot application, which logs in JSON format, using log4j2 library. [hello-metric](https://github.com/onkarshedge/hello-metric)[6]
> {"timeMillis":1605595134402,"thread":"http-nio-11000-exec-4","level":"INFO","loggerName":"com.example.hellometric.filter.LoggingFilter","message":"request completed successfully","endOfBatch":false,"loggerFqcn":"org.apache.logging.slf4j.Log4jLogger","contextMap":{"address":"0:0:0:0:0:0:0:1","method":"GET","path":"/greeting","referer":"null","request-id":"c9c9460f-69a7-4317-a371-59a1c2f5ce80","response-time":"1ms","status-code":"200"},"threadId":30,"threadPriority":5,"logTime":"17-11-2020T12:08:54 IST"}

This is in a single line, unpretty. Each log event needs be on a different line. If a single event, is split into multiple lines, they will appear as separate events in Splunk.

Install this app in my-app-dev namespace `kubectl apply -f deployment.yaml -n my-app-dev`.  

#### Searching logs in Splunk
Now visit splunk and search for the application logs.
{{< image src="/images/splunk_hello_metric_log.png" caption="Application log" >}}

##### Fields.conf {#fields-conf}
As we can see in the above image, We are able to search with `namespace` & `container_name`. While running splunk in a container, we had mounted this `fileds.conf` file.
```
[namespace]
INDEXED = true

[pod]
INDEXED = true

[container_name]
INDEXED = true

[container_id]
INDEXED = true

[cluster_name]
INDEXED = true
```

Hence, each event is enriched with the following fields `namespace, pod, container_name, cotainer_id, cluster_name`.  
Since, the log is structured, We can use `spath` command to search on nested fields. Here is a sample query to search for logs with non 200 status codes.  
> index="my_app" namespace="my-app-dev" container_name="hello-metric"  
  | spath output=statusCode "contextMap.status-code"   
  | search statusCode!=2*

We can click on any of the field value in log event and click `Add to search`. Splunk will write the query for us.

##### Searching k8s objects 
SCK forwards the k8s objects data in JSON to Splunk. Each change to k8s object is recorded as a separate event.
In values.yaml, we have given the list of kubernetes objects. In the below image, deployment objects are searched. We can see the entire object details metadata, spec, status etc.
{{< image src="/images/splunk_k8s_object.png" caption="K8s object log" >}}

Each object will have corresponding sourcetype as `kube:objects:<object_name>`

Here is a query to get all distinct image names with tag deployed in last week. 
`dedup` is distinct and will pick first entry sorted by time in ascending order.  
`table` table will show results in tabular form.

{{< image src="/images/splunk_unique_images.png" caption="Unique images deployed" >}}

#### Metrics
{{< admonition type=question title="What about the `my_app_metrics` that we created ?" open=true >}}
 As mentioned above daemonset.apps/splunk-splunk-kubernetes-metrics will send the metrics to Splunk server every 15s.
{{< /admonition >}}
Query to view available metrics:  
`| mcatalog values(metric_name) where index="my_app_metrics"`  
Query to view pod memory usage:  
`| mstats max(kube.pod.memory.usage_bytes) where index="my_app_metrics"`

#### Couple of Questions
{{< admonition type=question title="Why not log from application directly to Splunk ?" open=true >}}
log4j2 supports Splunk appender. If we log from application directly to Splunk, we won't see any logs with `kubectl logs <pod_name>`. As the container runtime handles logs to stdout & stderr.
Also, it tightly couples our application to Splunk.
{{< /admonition >}} 

{{< admonition type=question title="Why not use splunk logging-driver plugin for docker ?" open=true >}}
[https://docs.docker.com/config/containers/logging/splunk/](https://docs.docker.com/config/containers/logging/splunk/)
Kubernetes only supports json-file logging driver and is not configurable.
{{< /admonition >}}

#### Links
1) [https://splunk.github.io/docker-splunk/](https://splunk.github.io/docker-splunk/)
2) [https://docs.splunk.com/Documentation/SplunkCloud/8.1.2008/SearchReference/Commontimeformatvariables](https://docs.splunk.com/Documentation/SplunkCloud/8.1.2008/SearchReference/Commontimeformatvariables)
3) [https://docs.splunk.com/Documentation/Splunk/8.1.0/Knowledge/Usedefaultfields](https://docs.splunk.com/Documentation/Splunk/8.1.0/Knowledge/Usedefaultfields)
4) [https://github.com/splunk/splunk-connect-for-kubernetes/blob/main/helm-chart/splunk-connect-for-kubernetes/values.yaml](https://github.com/splunk/splunk-connect-for-kubernetes/blob/main/helm-chart/splunk-connect-for-kubernetes/values.yaml)
5) [https://docs.docker.com/config/containers/logging/configure/](https://docs.docker.com/config/containers/logging/configure/)
6) [https://github.com/onkarshedge/hello-metric](https://github.com/onkarshedge/hello-metric)



