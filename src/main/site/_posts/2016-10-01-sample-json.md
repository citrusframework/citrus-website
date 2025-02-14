---
layout: sample
title: Json sample
name: sample-json
image: /img/icons/json.png
folder: samples-json
group: validation
description: Shows Json payload validation feature with JsonPath validation
categories: [samples]
permalink: /samples/json/
---

This sample deals with Json message payloads when sending and receiving messages to the todo sample
application. Read about this feature in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides a REST API for managing todo entries.
We call this API and receive Json message structures for validation in our test cases.

We can use Json as message payloads directly in the test cases.
    
```java
http()
    .client(todoClient)
    .send()
    .post("/api/todolist")
    .messageType(MessageType.JSON)
    .contentType(ContentType.APPLICATION_JSON.getMimeType())
    .payload("{ \"id\": \"${todoId}\", \"title\": \"${todoName}\", \"description\": \"${todoDescription}\", \"done\": ${done}}");
```
        
As you can see we are able to send the Json data as payload. You can add test variables in message payloads. In a receive 
action we are able to use an expected Json message payload. Citrus performs a Json object comparison where each element is checked to meet
the expected values.

```java
http()
    .client(todoClient)
    .receive()
    .response(HttpStatus.OK)
    .messageType(MessageType.JSON)
    .payload("{ \"id\": \"${todoId}\", \"title\": \"${todoName}\", \"description\": \"${todoDescription}\", \"done\": ${done}}");
```

The Json message payload can be difficult to read when used as String concatenation. Fortunately we can also use file resources as message
payloads.

```java
http()
    .client(todoClient)
    .receive()
    .response(HttpStatus.OK)
    .messageType(MessageType.JSON)
    .payload(new ClassPathResource("templates/todo.json"));    
```
        
An alternative approach would be to use JsonPath expressions when validating incoming Json messages.

```java
http()
    .client(todoClient)
    .receive()
    .response(HttpStatus.OK)
    .messageType(MessageType.JSON)
    .validate("$.id", "${todoId}")
    .validate("$.title", "${todoName}")
    .validate("$.description", "${todoDescription}");
```
        
Each expression is evaluated and checked for expected values. In case a JsonPath expression can not be evaluated or 
does not meet the expected value the test ends with failure.    
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#validation-json
