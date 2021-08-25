---
layout: post
title: Apache Camel support in Citrus 3
short-title: Citrus and Apache Camel
author: Christoph Deppisch
github: christophd
categories: [blog]
---
 
Citrus 3 provides enhanced support for [Apache Camel](https://camel.apache.org) with new features and improvements.
This enables great opportunities and leverages the full enterprise integration power of Apache Camel in Citrus.

Before this post explains the enhancements in Citrus 3 regarding Camel let us have a brief look at what has always been in the box.

# Recap the Camel support in Citrus

Camel has been integrated with the Citrus framework for years. 
You can read about it in the official Citrus [user guide](https://citrusframework.org/citrus/reference/3.0.0/html/index.html#apache-camel).

With Camel included in the box Citrus users are able to use the 300+ Camel components in order to connect with various 
message transports and technologies.

Citrus is able to interact with Camel in following areas
                                                                                                           
* [Create/start/stop Camel routes](https://github.com/citrusframework/citrus/blob/main/src/manual/endpoint-camel.adoc#camel-route-actions)
* [Send/receive data to/from Camel routes](https://github.com/citrusframework/citrus/blob/main/src/manual/endpoint-camel.adoc#camel-endpoint)
* [Use Camel endpoint URI to produce messages via Camel endpoint components](https://github.com/citrusframework/citrus/blob/main/src/manual/endpoint-component.adoc)
* [Access the Camel control bus](https://github.com/citrusframework/citrus/blob/main/src/manual/endpoint-camel.adoc#camel-controlbus-actions)
* [Verify Camel exception handling](https://github.com/citrusframework/citrus/blob/main/src/manual/endpoint-camel.adoc#camel-exception-handling)

Usually Camel routes live in a Camel context defined in the Citrus project.

```java
@Bean
public CamelContext camelContext() {
    CamelContext context = new DefaultCamelContext();

    context.addRoutes(new RouteBuilder() {
        @Override
        public void configure() throws Exception {
            from("direct:hello")
                .routeId("helloRoute")
                .to("log:com.consol.citrus.camel?level=INFO")
                .to("seda:greetings");
        }
    });

    return context;
}
```

The context defines a Camel route listening on the endpoint uri `direct:hello`.
Incoming messages will be logged to the console using a `log` Camel component.
After that the message is forwarded to a `seda` Camel component which is a simple queue in memory.

The Citrus endpoint can interact with this sample route definition sending messages to the `direct:hello` endpoint.

```java
send("camel:direct:hello")
    .message()
    .body("Hello from Citrus!");
```

The Camel endpoint uri can refer to any Camel endpoint component.
See the list of available [Camel components](https://camel.apache.org/components/latest/index.html) in order to connect 
your route with message transports and technologies.

As an alternative to sending messages directly to a Camel endpoint uri you can create a Citrus endpoint that interacts with a Camel route. 
The endpoint can be shared in multiple tests and defines several properties such as the endpoint uri that will be called.
   
```java
@Bean
public CamelEndpoint helloCamelEndpoint() {
    return new CamelEndpoint()
            .endpointUri("direct:hello")
            .build();
}
```

```java
send(helloCamelEndpoint)
    .message()
    .body("Hello from Citrus!");
```

Of course, you can also consume messages from a Camel route using the endpoint pattern.

```java
receive("camel:seda:greetings")
    .message()
    .type(MessageType.PLAINTEXT)
    .body("Hello from Citrus!");
```

The Camel support in Citrus has been enhanced with Citrus 3. 
Read the following sections to find out about awesome new features and improvements.
              
# What's new in Citrus 3?

## Endpoint DSL support

Since Camel 3 you can use a Java fluent API to deal with Camel endpoint URIs. 
You can use the Camel endpoint DSL in the Citrus send/receive actions now, too.

```java
send(camel().endpoint(direct("hello")::getUri))
    .message()
    .body("Hello from Citrus!");

receive(camel().endpoint(seda("greetings")::getUri)")
    .message()
    .type(MessageType.PLAINTEXT)
    .body("Hello from Citrus!");
```

The send and receive actions above use the Camel endpoint DSL to construct the proper endpoint URI.
The `camel().endpoint(seda("test")::getUri)` builds the endpoint uri `seda:test`.
The endpoint DSL provides all settings and properties that you can set for a Camel endpoint component.

## Camel processor support

Camel implements the concept of processors as enterprise integration pattern.
A processor is able to add custom logic to a Camel route.
Each processor is able to access the Camel exchange that is being processed in the current route.
The processor is able to change the message content (body, headers) as well as the exchange information.

The send/receive operations in Citrus also implement the processor concept.
With the Citrus Camel support you can now use the very same Camel processor also in a Citrus test action.

```java
import com.consol.citrus.camel.message.CamelMessageProcessor;

public class CamelMessageProcessorIT extends TestNGCitrusSpringSupport {

    @Autowired
    private CamelContext camelContext;

    @Test
    @CitrusTest
    public void shouldProcessMessages() {
        CamelMessageProcessor.Builder toUppercase = camel(camelContext)
                .process(exchange -> exchange
                        .getMessage()
                        .setBody(exchange.getMessage().getBody(String.class).toUpperCase()));

        $(send(camel().endpoint(seda("test")::getUri))
                .message()
                .body("Citrus rocks!")
                .process(toUppercase)
        );
    }
}
```

The example above uses a Camel processor to change the exchange and the message content before the message is sent to the endpoint.
This way you can apply custom changes to the message as part of the test action.

```java
public class CamelMessageProcessorIT extends TestNGCitrusSpringSupport {

    @Autowired
    private CamelContext camelContext;

    @Test
    @CitrusTest
    public void shouldProcessMessages() {
        CamelMessageProcessor.Builder toUppercase = camel(camelContext)
                .process(exchange -> exchange
                        .getMessage()
                        .setBody(exchange.getMessage().getBody(String.class).toUpperCase()));
        
        $(send(camel().endpoint(seda("test")::getUri))
                .message()
                .body("Citrus rocks!"));

        $(receive(camel().endpoint(seda("test")::getUri))
                .process(toUppercase)
                .message()
                .type(MessageType.PLAINTEXT)
                .body("CITRUS ROCKS!"));
    }
}
```

The Camel processors are very powerful.
In particular, you can apply transformations of multiple kind.

```java
public class CamelTransformIT extends TestNGCitrusSpringSupport {

    @Autowired
    private CamelContext camelContext;

    @Test
    @CitrusTest
    public void shouldTransformMessageReceived() {
        $(send(camel().endpoint(seda("hello")::getUri))
                .message()
                .body("{\"message\": \"Citrus rocks!\"}")
        );

        $(receive(camel().endpoint(seda("hello")::getUri))
                .transform(
                    camel()
                        .camelContext(camelContext)
                        .transform()
                        .jsonpath("$.message"))
                .message()
                .type(MessageType.PLAINTEXT)
                .body("Citrus rocks!"));
    }
}
```

The transform pattern is able to change the message content before a message is received/sent in Citrus.
The example above applies a JsonPath expression as part of the message processing.
The JsonPath expression evaluates `$.message` on the Json payload and saves the result as new message body content.
The following message validation expects the plaintext value `Citrus rocks!`.

The message processor is also able to apply a complete route logic as part of the test action.

```java
public class CamelRouteProcessorIT extends TestNGCitrusSpringSupport {

    @Autowired
    private CamelContext camelContext;

    @Test
    @CitrusTest
    public void shouldProcessRoute() {
        CamelRouteProcessor.Builder beforeReceive = camel(camelContext).route(route ->
                route.choice()
                    .when(jsonpath("$.greeting[?(@.language == 'EN')]"))
                        .setBody(constant("Hello!"))
                    .when(jsonpath("$.greeting[?(@.language == 'DE')]"))
                        .setBody(constant("Hallo!"))
                    .otherwise()
                        .setBody(constant("Hi!")));

        $(send(camel().endpoint(seda("greetings")::getUri))
                .message()
                .body("{" +
                        "\"greeting\": {" +
                            "\"language\": \"EN\"" +
                        "}" +
                      "}")
        );

        $(receive("camel:" + camel().endpoints().seda("greetings").getUri())
                .process(beforeReceive)
                .message()
                .type(MessageType.PLAINTEXT)
                .body("Hello!"));
    }
}
```

With the complete route logic you have the full power of Camel ready to be used in your send/receive test action.
This enables many capabilities as Camel implements the enterprise integration patterns such as split, choice, enrich and many more.

## Camel data format support

Camel uses the concept of data format to transform message content in form of marshal/unmarshal operations.
You can use the data formats supported in Camel in Citrus, too.

```java
public class CamelDataFormatIT extends TestNGCitrusSpringSupport {

    @Autowired
    private CamelContext camelContext;

    @Test
    @CitrusTest
    public void shouldApplyDataFormat() {
        when(send(camel().endpoint(seda("data")::getUri))
                .message()
                .body("Citrus rocks!")
                .transform(camel(camelContext)
                        .marshal()
                        .base64())
        );

        then(receive("camel:" + camel().endpoints().seda("data").getUri())
                .transform(camel(camelContext)
                        .unmarshal()
                        .base64())
                .transform(camel(camelContext)
                        .convertBodyTo(String.class))
                .message()
                .type(MessageType.PLAINTEXT)
                .body("Citrus rocks!"));
    }
}
```

The example above uses the `base64` data format provided in Camel to marshal/unmarshal the message content to/from a base64 encoded String.
Camel provides support for many data formats as you can see in the [documentation on data formats](https://camel.apache.org/components/latest/dataformats/index.html).

## Wrap up!

That completes the new features in Citrus 3 regarding Apache Camel.

Please give it a try, provide some feedback and tell us what you think about Camel and Citrus!
