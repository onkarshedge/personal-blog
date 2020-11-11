---
title: "JSON Test"
date: 2020-11-10T11:45:37+05:30
draft: false
categories: ["testing"]
tags: ["java"]
---

##### Integration test, JSON assert

{{< admonition type=information title="Versions used" open=true >}}
* Java: 11
* Spring boot: 2.2.2
* JSON Unit: json-unit-spring:2.19.0, json-unit:2.19.0 
{{< /admonition >}}

We have a Spring boot application. We have @SpringBootTest integration tests for our application.
In integration tests we are asserting on the JSON response from the controller.
``` java
resultActions
             .andDo(print())
             .andExpect(content().json(expectedResponse));
```  
Problems with MockMVCResultMatchers json assertion:
1)  We do not want to assert on the value of ids and any other dynamically generated fields. But we do want to assert that they
are present in the response.
    ``` java
    result
          .andExpect(jsonPath("$.id", notNullValue()))
          .andExpect(jsonPath("$.name", is("user")));
    ```
    So we can use jsonPath, but it is not very convenient for large json responses. We would have to split the json file to assert or ignore nested fields if we go via jsonpath approach.
We store our json response in test/resources in a single file. 
2) We not only have 1 id field but other dynamic fields like certificateNumber, tradeSet etc. not necessarily at the top-level.

##### Solution: JSON Unit library

```java
resultActions
        .andDo(print())
        .andExpect(json().when(IGNORING_ARRAY_ORDER).isEqualTo(expectedResponse));
```
The code remains same, thanks to JSON unit it uses ResultMatcher and is easily pluggable.  
Now in our json file we can add id and other dynamic fields. We can assert id is present in the response, and it is a number.
```json
{
  "id": "${json-unit.any-number}",
  "referenceId": "${json-unit.any-number}",
  "certificateNumber": "${json-unit.any-number}",
  "referenceData": {
    "employeeIdsInTrade": [
      12345,
      6789
    ],
    "ActivityType": "STOCK_OPTIONS",
    "effectiveDate": "2020-05-16"
  },
  "description": "Stock Options Transaction Submission",
  "status": "PENDING_APPROVAL",
  "submittedBy": "john_doe",
  "submittedOn": "${json-unit.any-string}",
  "reviewedBy": null,
  "reviewedOn": null
}
```
The library is rich in features like, to list some:
* Jsonpath support
* Ignoring values
* Ignoring paths
* Regex
* Custom matchers

More on [https://github.com/lukas-krecan/JsonUnit#options](https://github.com/lukas-krecan/JsonUnit#options)



