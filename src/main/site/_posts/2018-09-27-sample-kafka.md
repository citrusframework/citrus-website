---
layout: sample
title: Kafka sample
name: sample-kafka
group: none
description: Shows Kafka integration
categories: [samples]
permalink: /samples/kafka/
---

This sample uses Kafka topics in order to place new todo entries in the system under test. The Kafka capabilities are
also described in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides a Kafka inbound topic listener for adding new todo entries.
We can send JSON messages in order to create new todo entries that are stored to the in memory storage.

The Citrus project needs a Kafka server that is provided as embedded server cluster. The embedded server is a singleton embedded Zookeeper
server and a single Kafka server with some test topics provided. The Kafka infrastructure is automatically started
with the tests. So the Citrus Kafka producer endpoint just needs to connect to the Kafka server broker.

```java
@Bean
public EmbeddedKafkaServer embeddedKafkaServer() {
    return new EmbeddedKafkaServerBuilder()
            .kafkaServerPort(9092)
            .topics("todo.inbound")
            .build();
}
    
@Bean
public KafkaEndpoint todoKafkaEndpoint() {
    return CitrusEndpoints.kafka()
            .asynchronous()
            .server("localhost:9092")
            .topic("todo.inbound")
            .build();
}
```

The endpoint connects to the server cluster and uses the topic `todo.inbound`. We can now place new todo entries to that topic in our test.
    
```java
send(todoKafkaEndpoint)
        .header(KafkaMessageHeaders.MESSAGE_KEY, "${todoName}")
        .payload("{ \"title\": \"${todoName}\", \"description\": \"${todoDescription}\" }");
```
        
We can add a special message header **KafkaMessageHeaders.MESSAGE_KEY** which is the Kafka producer record message key. The message key is automatically serialized/deserialized as String value. 
The Kafka record value is also defined to be a String in JSON format. The todo app will consume the Kafka record and create a proper todo entry from that record.

The Kafka operation is asynchronous so we do not get any response back. Next action in our test deals with validating that the new todo 
entry has been added successfully. Please review the next steps in the sample test that perform proper validation.
        
Run
---------

**NOTE:** This test depends on the [todo-app](/samples/todo-app/) WAR which must have been installed into your local maven repository using `mvn clean install` beforehand.

The sample application uses Maven as build tool. So you can compile, package and test the
sample with Maven.
 
     mvn clean verify -Dembedded
    
This executes the complete Maven build lifecycle. The embedded option automatically starts a Jetty web
container before the integration test phase. The todo-list system under test is automatically 
deployed in this phase. After that the Citrus test cases are able to interact with the todo-list application in the integration test phase.

During the build you will see Citrus performing some integration tests.
After the tests are finished the embedded Jetty web container and the todo-list application are automatically stopped.

Read further the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#kafka
