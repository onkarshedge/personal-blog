---
title: "Splunk Logging K8s"
date: 2020-06-25T11:44:46+05:30
draft: true
---
Splunk running in docker: 8.0.3  
Docker for desktop: 2.3.0.2  
Kubernetes: v1.16.5  
Docker Engine: 19.03.8  
OS: Catalina 10.15.4  

Terminologies:
* Source type: Type of data, so that Splunk format intelligently while indexing
* App context: Folders within Splunk for managebility and separation of domain, configuration.
* Index: reside in flat files on the Splunk platform instance known as the indexer. Splunk transforms the data into searchable events and stores those events in the index.
Consider creating multiple indexes based on different access privileges, retention periods, or logical groupings.

Run Splunk in container.  
``docker run --name splunk -d -v `pwd`/splunk_data:/splunk_data -v `pwd`/fields.conf:/opt/splunk/etc/system/local/fields.conf -p 8000:8000 -p 8089:8089 -p 8088:8088 -e 'SPLUNK_START_ARGS=--accept-license' -e 'SPLUNK_PASSWORD=splunk_admin' splunk/splunk:latest
``

My local directory splunk_data has a earthquakes.csv file, which I am mounting so that I can add it as a file source in Splunk.
There are 2 options to continuously monitor the directory or index once.
1) Adding Data
    * Set soucetype CSV
    * Timestamp fields: Datetime
    * TIME_FORMAT: %A, %B %d, %Y %H:%M:%S %Z
    * MAX_DAYS_AGO: 10000 since the data is from 2013
    * Save in which ever app you want.
    * Specify a new index.
    
    #### \<World Map Dashboard here>
    ``index="mcap" sourcetype="csv" | geostats latfield=Lat longfield=Lon count``
    `` | makeresults | eval ip="34.98.204.43" | iplocation ip | geostats latfield=lat longfield=lon count``
    ``| makeresults | eval ip="34.98.204.43" | append [| makeresults | eval ip="103.94.56.63" ] | iplocation ip | geostats latfield=lat longfield=lon count``
    ``index="mcap" sourcetype="csv" 
      | lookup geo_us_states longitude as Lon, latitude as Lat 
      | stats count by featureId
      | geom geo_us_states``
Now let's get the log of our application running in kubernetes.
    
2) Generating HEC(Http Event Collector token)
    * Settings > Data Inputs > HEC. This token is used to forward data to Splunk server over http. It serves as a purpose of authentication. HEC uses the source, source type, and index that was specified in the token.
    * Specify separate indices for logging, objects, monitoring
    * In Data Inputs > HTTP Event Collector you will see your token.
    
3) I have two namespaces monitoring, my-app-dev
    * kubectl create ns my-app-dev
    * kubectl create ns monitoring
    * Optional to install dashboard 
    ```
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc3/aio/deploy/recommended.yaml
    Visit http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
    kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | awk '/^deployment-controller-token-/{print $1}') | awk '$1=="token:"{print $2}'
    ```

4) SCK(Get sample values.yaml for all 3 components from the github repo.)
    * Replace HEC token and host accordingly.   
``helm install splunk --namespace=monitoring -f sck_values.yaml https://github.com/splunk/splunk-connect-for-kubernetes/releases/download/1.4.1/splunk-connect-for-kubernetes-1.4.1.tgz
``  
Show the objects created.

minikube ssh equivalent for docker desktop is 
``docker run -it --privileged --pid=host justincormack/nsenter1``
Now you should be able to see fluentd running with ps -a | grep fluentd

``kubectl apply -f app_deployment.yaml``
Show logs on Splunk 

``| mcatalog values(metric_name) where index="mcap_metrics"``
``| mstats max(kube.pod.memory.usage_bytes) where index="mcap_metrics" by "pod"``
