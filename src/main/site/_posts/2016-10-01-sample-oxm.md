---
layout: sample
title: Object marshalling sample
name: sample-oxm
icon: code
folder: samples-xml
group: validation
description: Shows XML object marshalling feature when sending and receiving messages
categories: [samples]
permalink: /samples/oxm/
---

This sample demonstrates the usage of object mapping in Citrus. We are able to handle automatic object mapping
when sending and receiving message payloads. Read about this feature in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides a REST API for managing todo entries.
We call this API with object mapping in Citrus so that we do not need to write message payload JSON or XML
structures but use the model objects directly in our test cases.

We need to include the Spring oxm module in the dependencies:

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-oxm</artifactId>
  <version>${spring.version}</version>
  <scope>test</scope>
</dependency>
```
    
Also we need to provide a marshaller component in our Spring configuration:
    
```java
@Bean
public Marshaller marshaller() {
    Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
    marshaller.setContextPath("com.consol.citrus.samples.todolist.model");
    return marshaller;
}
```
    
Please note that the marshaller supports model object classes in package **com.consol.citrus.samples.todolist.model**. 

That is all for configuration, now we can use model objects as message payload in the test cases.
    
```java
@Autowired
private Jaxb2Marshaller marshaller;
    
http()
    .client(todoClient)
    .send()
    .post("/api/todolist")
    .contentType(ContentType.APPLICATION_JSON.getMimeType())
    .payload(new TodoEntry("${todoName}", "${todoDescription}"), marshaller);
```
        
As you can see we are able to send the model object as payload. The test variable support is also given. Citrus will automatically marshall the object to a **application/json** message content 
as **POST** request. In a receive action we are able to use a mapping validation callback in order to get access to the model objects of an incoming message payload.

```java
http()
    .client(todoClient)
    .receive()
    .response(HttpStatus.OK)
    .validationCallback(new XmlMarshallingValidationCallback<TodoEntry>(marshaller) {
        @Override
        public void validate(TodoEntry todoEntry, Map<String, Object> headers, TestContext context) {
            Assert.assertNotNull(todoEntry);
            Assert.assertEquals(todoEntry.getId(), uuid);
        }
    });
```
        
The validation callback gets the model object as first method parameter. You can now add some validation logic with assertions on the model object.    
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#validation-callback
