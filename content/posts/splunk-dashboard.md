---
title: "Splunk Dashboard"
date: 2020-11-20T11:25:37+05:30
draft: false
tags: ["logging"]
categories: ["splunk"]
---

### Splunk Dashboard

Visualization helps us see information in new ways. We can detect a trend or anomalies in the data. How is log data useful for visualization ?.
We can extract information from INFO, ERROR logs. Structured logs(JSON, key/value pairs) make it easy to extract information.
This data can be visualized over a period to understand trends, anomalies. It can help us to take actions(eg. creating alerts) or answer our OKR(Objectives Key Requirements). 
 
Splunk dashboard is easy to create. We just to have to write the search queries in SPL(Splunk Processing language) and choose a suitable visualization. No HTML, CSS knowledge required.
The visualizations are customizable, we can change colors, X-Y axes labels, threshold colors based on numbers etc.

### Query format

Splunk Query follows the Unix Philosophy of piping. First comes the search query which is `key=value` syntax. Followed by pipe `|` operator, search command or aggregate command, and its respective parameters.
> index="my_app" namespace="dev" level=ERROR    
> | <command_1> <command_param>  
> | <command_2> <command_2_param>

It can also start with `| search`
### Input Controls

Time is the most important/necessary filter. We would want to view the dashboard over a quarter for a summary or narrow down to a particular day/hour to understand a problem.
We will also need other filters like Environment, Service, Region etc.  
Splunk provides the following input controls:  
* check box
* dropdown
* link list
* radio
* multiselect

The dropdown option values can be static i.e DEV -> "k8s-app-dev", QA -> "k8s-app-qa" or they can be dynamic via a search query.
In the latter case, we would need to specify the field returned from the search result. Each dashboard panel has it's own query. The value can then be used in search query as `$token$`.
eg. `| search namespace=$env$`. Time filter can be specified to use the shared time filter or a custom private one.  
The dashboard will refresh on every input/time change, or it can be set to auto-refresh.

### Single Stat Panel
This looks good at the top of the dashboard. As it gives a summary of key numbers. The four individual panels are aligned in a row.
They can be resized and aligned however we like. The color can be configured based on a range of threshold values. eg. 0-50 -> green, 50-100 -> orange etc.

{{< image src="/images/splunk_single_stat.png" caption="Single Stat">}}

``` Splunk Query
index="my_app" thrown.message"="User not found *" container_name="usermanagement" namespace="$namespace$"
 | rex field="thrown.message" "User not found (?<emailID>.*)"
 | stats count
```

> **rex** : rex is used to capture matching regex pattern into a field. Above we are capturing emailId from the error message into field: `emailID`  
> **stats** : stats is an aggregate function, we can do count, avg, min, max etc.

### Table
As the name suggests, we can choose certain fields to be shown in table format. Below is a table showing user's last login time. We can take action to revoke access or send an email alert.

{{< image src="/images/splunk_table.png" caption="Last Login time">}}

``` Splunk Query
| spath output=status "contextMap.status-code"
| spath output=path "contextMap.path"
| search path=/api/v1/users/login status=2*
| dedup user sortby -_time
| eval lastLoginTime=strftime(_time, "%x %I:%M:%S %p")
| eval daysSince = floor((now() - _time) / 86400)
| sort -daysSince
| rename user as "User ID", lastLoginTime as "Last Login Time", daysSince as "Days Since Last Login"
| table "User ID", "Last Login Time", "Days Since Last Login"
```

> **spath** : To extract a nested JSON/XML field into an event field.  
> **dedup** : Get distinct events by user field. By default, it will take the first field sorted by time in ascending order.
Here we are specifying descending order with `-` symbol. `_time` is a default field available in each event.    
> **eval** : To create a derived field using exiting fields, or it can be a constant too.  
> **rename** : The fields chosen in `table` command are the column header. To show a user friendly name, we can rename them.  
> **table**: To show a table view with selected fields. 

### PunchCard
Punch Card is used to view a data across two dimensions. Here we are having time and day of the week as 2-dimensions.
The count of http requests is the bubble. The size of the bubble is relative.  
There was no usage on Sunday's and peak usage on Thursday, Friday between 12AM to 2AM.

{{< image src="/images/splunk_punchcard.png" caption="Peak Usage">}}

``` Splunk Query
index="my-app" container_name="gateway"
| spath output=path "contextMap.path"
| regex "path"="^((?!actuator).)*$"
| spath output=status "contextMap.status-code"
| where isnotnull(status)
| stats count by date_hour date_wday
```
Every request from the user goes via gateway. Hence, the above query is counting the number of http requests. 

> **regex** : Regex is used to filter events whereas rex is used to capture matching patterns. We are filtering out actuator requests,
as the kubelet hits this path periodically for health check.    
> **date_hour, date_wday** : Default fields available

### Line Chart
Used for single or multiple data-series. 

{{< image src="/images/splunk_line_chart_single.png" caption="Single Line Chart">}}

{{< image src="/images/splunk_line_chart_multiple.png" caption="Multiple Line Chart">}}

``` Splunk Query
mcap_idx` container_name="finance_app" namespace="$namespace$"
| spath output=status "contextMap.status-code"
| spath output=method "contextMap.method"
| spath output=path   "contextMap.path"
| search status="2*" method!=GET (path="/api/v*/transactions/*" OR path="/api/v*/bulkupload/transactions/process/*")
| eval transaction_type = case(match(path, "/api/v\d{1}/transactions/shares/buy"), "Share Buy",
    match(path, "/api/v\d{1}/transactions/shares/sell"), "Share Sell",
    match(path, "/api/v\d{1}/transactions/X+/settlement"), "Settlement",
    match(path, "/api/v\d{1}/transactions/settlements"), "Bulk Settlement",
    match(path, "/api/v\d{1}/bulkupload/transactions/process/X+"), "Bulk Transactions",
    1 == 1, path)
| timechart span=1$span$ count(eval(match(transaction_type, if("$txn_type$"=="all",".*","$txn_type$")))) AS txn_cnt
```

> **case** : Switch case statement.  
> **match** : To match a particular field on a regex pattern. 1==1 is used as a default case statement.    
> **timechart**: `| timechart span=1h count by <field_name>`. Is what you will find in most examples. The timechart expression in 
the above query is complex because, I had a `ALL` option in the dropdown input too.  
> **span**: the span option for timechart is the x-axis time interval. It can be days, months, weeks, hours etc. I have it configurable via a dropdown input token.

### Bar chart

Alternative visualization for line chart is a bar chart or a stacked bar chart.

{{< image src="/images/splunk_stack_bar_chart.png" caption="stacked bar chart">}}

> The query will be same like line chart `| timechart span="1$span$" count by approvalType` only visualization is changed.

Every commit on master is pushed to dev.
{{< image src="/images/splunk_bar_chart_dev.png" caption="No. of deployments on dev">}}

Production release is done weekly.
{{< image src="/images/splunk_bar_chart_prod.png" caption="No. of deployments on prod">}}

``` Splunk Query
index="app-container" sourcetype IN ("kube:objects:events:watch") "object.involvedObject.kind"=Deployment type=ADDED
| spath output=ns "object.involvedObject.namespace"
| spath output=uid "object.metadata.uid"
| spath output=msg object.message
| spath output=service "object.involvedObject.name"
| spath output=firstTime "object.firstTimestamp"
| search ns=$namespace$ msg="Scaled up replica set*"
| eval eventTime=strptime(firstTime,"%Y-%m-%dT%H:%M:%S")
| rex field=msg "Scaled up replica set (?<replicaName>[^\s]+).*"
| dedup replicaName sortby -eventTime
| timechart span=1$span$ count by service
```

> Every time a deployment is updated, a new replica set is created and scaled up to the desired count. I am searching for logs of type
kube:objects:events:watch and involvedObject is deployement. If the number of replicas specified is 3 and it is a rolling deployment. There will be 3 events.    
> Scaled up replica set my-app-3423 to 1*  
> Scaled up replica set my-app-3423 to 2*  
> Scaled up replica set my-app-3423 to 3*  
>
> Hence, deduping on replicaName.

### Pie Chart
Again, the query is same `| stats count by <fieldname>`

{{< image src="/images/splunk_pie_chart.png" caption="Pie Chart">}}


### Maps
This one is my favourite. There are two map visualizations available Cluster Map, Choropleth Map.

#### Cluster Map

Splunk has a command `iplocation`. eg. `| iplocation <ip_field_name>`. It will add City, Country, lat, lon, and Region fields to each event.
Splunk ships with a Geolite2-City.mmdb database. GeoLite2 databases are free IP geolocation databases. The paid ones are more accurate.

{{< image src="/images/splunk_ip_location.png" caption="iplocation">}}

> **makeresults** : Generate some events, takes count as a parameter.  
> **makemv**: Convert an existing field into a multivalue field. Ipaddress field was a string. It is split by comma
and converted into a multivalue field.  
> **mvexpand**: Expands the single event into multiple events for each value.    
> **fields**: Used to select or discard certain fields. `-` to discard.

{{< image src="/images/splunk_geostats.png" caption="Geo stats map">}}

> `geostats` command is used to plot this map. The size of the circles is comparative. 
> We can count by country, region, city etc.

#### Choropleth Map

Sometime we want the states/countries to be colored on a linear scale or categorically. Choropleth can be used here.

{{< image src="/images/splunk_choropleth_world.png" caption="Choropleth map">}}

> I am generating 3500 events. Distributing 4 ipaddresses into buckets. In reality, we can directly use the data.  
> **rangemap** : assign a categorical value to a numeric field.  
> **geomstats** : Creates a choropleth map (categorical or linear)


##### USA Presidential Elections 2020

Here is a categorical map of USA Presidential elections 2020.

{{< image src="/images/splunk_usa.png" caption="USA Elections">}}

<!--

-->













   
