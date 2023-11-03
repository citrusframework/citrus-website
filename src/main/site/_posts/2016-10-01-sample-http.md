---
layout: sample
title: Http REST sample
name: sample-http
group: samples-http
description: Shows REST API calls as a client
categories: [samples]
permalink: /samples/http/
---

This sample demonstrates the Http REST capabilities in Citrus where Citrus calls REST API on a todo web application. REST features are
also described in detail in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides a REST API for managing todo entries.
Citrus is able to call the API methods as a client in order to validate the Http response messages.

We need a Http client component in the configuration:

```java
@Bean
public HttpClient todoClient() {
    return CitrusEndpoints.http()
                        .client()
                        .requestUrl("http://localhost:8080")
                        .build();
}
```
    
In test cases we can reference this client component in order to send REST calls to the server.
    
```java
http()
    .client(todoClient)
    .send()
    .post("/todolist")
    .contentType("application/x-www-form-urlencoded")
    .payload("title=${todoName}&description=${todoDescription}");
```
        
As you can see we are able to send **x-www-form-urlencoded** message content as **POST** request. The response is then validated as **Http 200 OK**.

```java
http()
    .client(todoClient)
    .receive()
    .response(HttpStatus.OK);
```
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#http
