---
layout: sample
title: Java config sample
name: sample-javaconfig
group: none
description: Uses pure Java POJOs for configuration
categories: [samples]
permalink: /samples/javaconfig/
---

This sample uses pure Java POJOs as configuration.

Objectives
---------

Citrus uses Spring Framework as glue for everything. Following from that Citrus components are
defined as Spring beans in an application context. You can use XML
configuration files and you can also use Java POJOs.

This sample uses pure Java code for both Citrus configuration and tests. The
Citrus TestNG test uses a context configuration annotation.

```java
@ContextConfiguration(classes = { EndpointConfig.class })
```
    
This tells Spring to load the configuration from the Java class ***EndpointConfig***.
    
```java
@Bean
public HttpClient todoClient() {
    return CitrusEndpoints.http()
                .client()
                .requestUrl("http://localhost:8080")
                .build();
}
```
    
In the configuration class we are able to define Citrus components for usage in tests. As usual
we can autowire the Http client component as Spring bean in the test cases.
    
```java
@Autowired
private HttpClient todoClient;
```
    
As usual we are able to reference this endpoint in any send and receive operation in Citrus Java fluent API.

```java
http()
    .client(todoClient)
    .send()
    .get("/todolist")
    .accept("text/html");
```
        
Citrus and Spring framework automatically injects the endpoint with respective configuration for `requestUrl = http://localhost:8080`.    
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#validation-xhtml
