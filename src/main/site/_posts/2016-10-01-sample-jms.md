---
layout: sample
title: JMS sample
name: sample-jms
image: /img/icons/activemq.png
folder: common
group: endpoints
description: Shows JMS message broker connectivity
categories: [samples]
permalink: /samples/jms/
---

This sample uses JMS queue destinations in order to place new todo entries in the system under test. The JMS capabilities are
also described in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides a JMS inbound message listener for adding new todo entries.
We can send JSON messages in order to create new todo entries that are stored to the in memory storage.

The Citrus project needs a JMS connection factory that is defined in the Spring application context as bean:

```java
@Bean
public ConnectionFactory connectionFactory() {
    return new ActiveMQConnectionFactory("tcp://localhost:61616");
}
```
    
We use ActiveMQ as message broker so we use the respective connection factory implementation here. The message broker is automatically
started with the Maven build lifecycle.

We can use that connection factory in a JMS endpoint configuration:

```java
@Bean
public JmsEndpoint todoJmsEndpoint() {
    return CitrusEndpoints.jms()
            .asynchronous()
            .connectionFactory(connectionFactory())
            .destination("jms.todo.inbound")
            .build();
}
```

The endpoint defines the connection factory and the JMS destination. In our example this is a message queue name `jms.todo.inbound`. JMS topics are also supported read about it in
[reference guide][1].    
    
No we can add a new todo entry by sending a JSON message to the JMS queue destination.
    
```java
send(todoJmsEndpoint)
    .header("_type", "com.consol.citrus.samples.todolist.model.TodoEntry")
    .payload("{ \"title\": \"${todoName}\", \"description\": \"${todoDescription}\", \"done\": ${done}}");
```
        
We have to add a special message header **_type** which is required by the system under test for message conversion. The message payload
is the JSON representation of a todo entry model object.

The JMS operation is asynchronous so we do not get any response back. Next action in our test deals with validating that the new todo 
entry has been added successfully. The XPath expression validation makes sure the the last todo entry displayed is the todo item that 
we have added before in the test.

You can read about http and XPath validation features in the sample [xhtml](../sample-xhtml/README.md)

In order to demonstrate the receive operation on a JMS queue in Citrus we can trigger a JMS report message on the todo-app server via Http.

```java
http()
    .client(todoClient)
    .send()
    .get("/api/jms/report/done")
    .accept(MediaType.APPLICATION_JSON_VALUE);

http()
    .client(todoClient)
    .receive()
    .response(HttpStatus.OK);
```

The Http GET request triggers a JMS report generation on the todo-app SUT. The report is sent to a JMS queue destination `jms.todo.report`. We can receive that message within Citrus
with a normal `receive` operation on a JMS endpoint.

```java
@Bean
public JmsEndpoint todoReportEndpoint() {
    return CitrusEndpoints.jms()
            .asynchronous()
            .connectionFactory(connectionFactory())
            .destination("jms.todo.report")
            .build();
}
```

```java
receive(todoReportEndpoint)
                .messageType(MessageType.JSON)
                .payload("[{ \"id\": \"${todoId}\", \"title\": \"${todoName}\", \"description\": \"${todoDescription}\", \"attachment\":null, \"done\":true}]")
                .header("_type", "com.consol.citrus.samples.todolist.model.TodoEntry");
```

The action receives the report message from that JMS queue and validates the message content (payload and header).
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. 

As we want to use the JMS capabilities of the too application we need to start the ActiveMQ message broker first. 
You can do this with Maven:
 
    mvn activemq:run

Read further the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#jms
