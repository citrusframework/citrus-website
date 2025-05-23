---
layout: sample
title: Apache Camel sample
name: sample-camel-context
image: /img/icons/camel.png
folder: common
group: demo
description: Interact with Apache Camel
categories: [samples]
permalink: /samples/apache-camel/
---

This sample demonstrates how Citrus is able to interact with Apache Camel. Read more about this in [reference guide][1]

Objectives
---------

Apache Camel project implements the enterprise integration patterns for building mediation and routing rules in your enterprise application. With the Citrus 
Camel support you are able to directly interact with the Apache Camel components and route definitions. You can call Camel routes and receive synchronous response messages. 
You can also simulate the Camel route endpoint with receiving messages and providing simulated response messages.

So we need a Camel route to test.

```java
@Bean
public CamelContext camelContext() throws Exception {
    SpringCamelContext context = new SpringCamelContext();
    context.addRouteDefinition(new RouteDefinition().from("jms:queue:JMS.Queue.News")
                                                .to("log:com.consol.citrus.camel?level=INFO")
                                                .to("spring-ws:http://localhost:18009?soapAction=newsFeed"));
    return context;
}
```

The Camel route reads from a JMS queue and forwards the message to a SOAP web service endpoint. In a test scenario we need to send messages to the JMS destination and wait for messages on
the SOAP server endpoint. Let's add configuration for this in Citrus:

```java
@Bean
public JmsEndpoint newsJmsEndpoint() {
    return CitrusEndpoints.jms()
            .asynchronous()
            .timeout(5000)
            .destination("JMS.Queue.News")
            .connectionFactory(connectionFactory())
            .build();
}

@Bean
public WebServiceServer newsServer() {
    return CitrusEndpoints.soap()
            .server()
            .autoStart(true)
            .timeout(10000)
            .port(18009)
            .build();
}
```

The components above are used in a Citrus test case.

```java
@Test
public class NewsFeedIT extends TestNGCitrusTestDesigner {

    @CitrusTest(name = "NewsFeed_Ok_IT")
    public void newsFeed_Ok_Test() {
        send("newsJmsEndpoint")
                .payload("<nf:News xmlns:nf=\"http://citrusframework.org/schemas/samples/news\">" +
                            "<nf:Message>Citrus rocks!</nf:Message>" +
                        "</nf:News>");

        receive("newsServer")
                .payload("<nf:News xmlns:nf=\"http://citrusframework.org/schemas/samples/news\">" +
                            "<nf:Message>Citrus rocks!</nf:Message>" +
                        "</nf:News>")
                .header(SoapMessageHeaders.SOAP_ACTION, "newsFeed");

        send("newsServer")
                .header(SoapMessageHeaders.HTTP_STATUS_CODE, "200");
    }
}
```

As you can see Citrus is both JMS message producer and SOAP server at the same time in a single test. The Apache Camel route in the middle will read the JMS message and forward it to the SOAP
server endpoint where Citrus receives the message for validation purpose. This way we make sure the Camel route is working as expected.    

Run
---------

The sample application uses Maven as build tool. So you can compile, package and test the
sample with Maven.

```shell
mvn install -Dembedded=true
``` 

This executes the complete Maven build lifecycle.

During the build you will see Citrus performing some integration tests.

Citrus test
---------

Execute all Citrus tests by calling

```shell
mvn integration-test
```

You can also pick a single test by calling

```shell
mvn integration-test -Ptest=TodoListIT
```

You should see Citrus performing several tests with lots of debugging output in your terminal. 
And of course green tests at the very end of the build.

Of course, you can also start the Citrus tests from your favorite IDE.
Just start the Citrus test using the TestNG IDE integration in IntelliJ, Eclipse or Netbeans.

 [1]: https://citrusframework.org/citrus/reference/html#camel
