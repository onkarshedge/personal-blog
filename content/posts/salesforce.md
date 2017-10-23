---
title: "Salesforce"
date: 2017-10-23T17:27:35+05:30
tags: ["salesforce","postman"]
draft: true
---

Explored the salesforce API. Got a developer salesforce account from someone. Initially I tried to use the rest API from python script, but realized it was time consuming to start writing code for oauth2 etc..

So I used Postman, in postman you can have oauth2 implementation where you can specify the urls.
* authorization url:
* token url:
* client_id:
* client_secret:
* grant-type:

The callback url of Postman is https://www.getpostman.com/oauth2/callback
Then you will be redirected to the UI of the application, login followed by asking to grant access and you get back a token.
<p>
There is developer console where the user can view the data and run queries.
The goal was to get familiar with Bulk API, as we wanted to get all the salesforce data. So at the time of this writing Bulk API 2.0 did not support query or queryAll, leaving us with 1.0 and in that I used v41.0</p>
The way you can get all the data from Salesforce is
* get the list of all objects
```
  curl https://{{yourInstance}}.salesforce.com/services/data/v37.0/sobjects/ -H "Authorization: Bearer token".  
  This will return a JSON response.
```
* for each object get the fields
```
  The JSON response of the sobject contains a urls field with "describe" key that has the url to the fields of this object.  
  "/services/data/v37.0/sobjects/Account/describe"
```
* Bulk API is as below
* create a job for that object
```
  curl https://{{instance_name}}—api.salesforce.com/services/async/{{APIversion}}/job  -H "X-SFDC-Session: token" -d  
  {
    "operation" : "query",
    "object" : "Account",
    "contentType" : "CSV"
}
weird is that bulk api does not have authorization header but this session whose value is token itself without the Bearer part
```
* add a batch operation to that job
```
  curl https://{{instance_name}}-api.salesforce.com/services/async/41.0/job/{{jobID}}/batch
  -H "X-SFDC-Session: token" -H "Content-Type: text/csv;charset=UTF-8" -d "select field from object"  
  The content type has to be the same which was set for the job, the format in which you want to receive the data.
```
* monitor the job and batch status
```
   https://{{instance_name}}-api.salesforce.com/services/async/41.0/job/{{jobID}}
   You will receive a response with batch failed, completed, number of records etc. jobID is received when job was created.  
   https://{{instance_name}}-api.salesforce.com/services/async/41.0/job/{{jobID}}/batch/{{batchID}}  
   batchID is received when batch was added to the job.
```
* get the result for each batch
```
  https://{{instance_name}}.salesforce.com/services/async/41.0/job/{{jobID}}/batch/{{batchID}}/result
  returns a response containing resultID
```
* from resultID get the actual data
```
  https://{{instance_name}}.salesforce.com/services/async/41.0/job/{{jobID}}/batch/{{batchID}}/result/{{resultID}}  
  will give the response containing all the data
```

In the curl commands if the token contains special character like ! it has to be escaped \! but not in postman.  
Also today I learn you can have environments in postman, add some variables and then use them in collection like urls, header values etc with {{}}. Exported and shared with team members
