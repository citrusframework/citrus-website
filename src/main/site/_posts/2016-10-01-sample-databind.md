---
layout: sample
title: Json databind sample
name: sample-databind
icon:  json
folder: samples-json
group: validation
description: Shows JSON object mapping feature when sending and receiving messages
categories: [samples]
permalink: /samples/databind/
---

This sample demonstrates the usage of object mapping in Citrus. We are able to handle automatic object mapping
when sending and receiving message payloads. Read about this feature in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides a REST API for managing todo entries.
We call this API with object mapping in Citrus so that we do not need to write message payload JSON or XML
structures but use the model objects directly in our test cases.

In test cases we can use the model objects directly as message payload.
    
```java
http()
    .client(todoClient)
    .send()
    .post("/api/todolist")
    .contentType(ContentType.APPLICATION_JSON.getMimeType())
    .payload(new TodoEntry("${todoName}", "${todoDescription}"), objectMapper);
```
        
As you can see we are able to send the model object. Citrus will automatically convert the object to a **application/json** message content 
as **POST** request. In a receive action we are able to use a mapping validation callback in order to get access to the model objects of an incoming message payload.

```java
http()
    .client(todoClient)
    .receive()
    .response(HttpStatus.OK)
    .validationCallback(new JsonMappingValidationCallback<TodoEntry>(TodoEntry.class, objectMapper) {
        @Override
        public void validate(TodoEntry todoEntry, Map<String, Object> headers, TestContext context) {
            Assert.assertNotNull(todoEntry);
            Assert.assertEquals(todoEntry.getId(), uuid);    
        }
    });
```
        
The validation callback gets the model object as first method parameter. You can now add some validation logic with assertions on the model.    
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#validation-callback
