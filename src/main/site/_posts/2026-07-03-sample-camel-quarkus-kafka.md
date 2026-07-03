---
layout: sample
title: Apache Camel - Quarkus Kafka Eventing
name: camel-kafka
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel Kafka routes in Quarkus with Citrus and Testcontainers
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-kafka/
---

Apache Camel is one of the most popular integration frameworks in the Java ecosystem, and when combined with Quarkus and Kafka it offers a powerful foundation for building event-driven microservices. 

But testing these integrations end to end remains a challenge. Camel routes consume and produce Kafka messages asynchronously, and verifying that the entire pipeline works correctly requires a real broker, proper topic setup, and a testing framework that understands messaging.

This post walks you through testing an Apache Camel Kafka route in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus provisions a Kafka broker with a single annotation, sends messages into a Camel route, and validates the output — all with clean, readable test code. By the end you will have a clear recipe for testing your own Camel Kafka routes in Quarkus.

# The application under test

The example application is intentionally simple so we can focus on the testing side. It is a Quarkus service that uses an Apache Camel route to consume text messages from a Kafka topic, transform them to uppercase, and publish the result to an output topic.

```
Kafka Topic (words-in) --> Apache Camel Route (Transform) --> Kafka Topic (words-out)
```

The entire integration logic lives in a single Camel route defined using the type-safe Endpoint DSL:

```java
public class Routes extends EndpointRouteBuilder {

    @Override
    public void configure() throws Exception {
        from(kafka("words-in").autoOffsetReset("earliest"))
            .setBody(exchange -> ">> " + exchange.getIn().getBody().toString().toUpperCase())
            .to(kafka("words-out"));
    }
}
```

The route reads messages from the `words-in` Kafka topic, transforms the body to uppercase with a `">> "` prefix, and sends the result to the `words-out` topic. The `autoOffsetReset("earliest")` setting ensures the consumer starts reading from the beginning of the topic, which is important for test reliability.

By extending `EndpointRouteBuilder` instead of the plain `RouteBuilder`, the route gains access to Camel's Endpoint DSL. This means `kafka("words-in")` is a type-safe method call with IDE auto-completion, not a string URI that could contain typos. Quarkus discovers this class automatically and starts the route when the application boots.

The production configuration is minimal — just the broker address in `application.properties`:

```properties
kafka.bootstrap.servers=localhost:9092
```

That is the entire application. Now let's verify it.

# Adding Citrus to the project

Citrus is highly modular, so you only pull in the capabilities you actually need. For this Camel Kafka scenario you add four test-scoped dependencies to the Maven POM:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-kafka</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-testcontainers</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-junit-jupiter</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Here is what each module brings to the table:

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle. It hooks into `@QuarkusTest` so that Citrus endpoints and test runners are available without any extra configuration.
- **citrus-kafka** provides Kafka endpoint implementations with built-in producer and consumer capabilities, message serialization, and topic configuration.
- **citrus-testcontainers** adds the `@KafkaContainerSupport` annotation that provisions a real Kafka broker for your tests. This is the key dependency for this example.
- **citrus-junit-jupiter** connects Citrus to JUnit 5, the test engine that Quarkus uses under the hood.

# Why not Quarkus dev services?

If you have read the [event-driven Kafka sample](/samples/quarkus-event-driven-kafka/) on this site, you might wonder why we are not relying on Quarkus Kafka dev services to start the broker. Dev services are great when your application uses SmallRye Reactive Messaging, because Quarkus knows exactly how to wire the broker connection to the messaging channels.

But when your application uses Apache Camel's native Kafka component, the situation is different. Camel manages its own Kafka consumer and producer configuration through the Camel Kafka component, and Quarkus dev services may not automatically configure that connection. You need a way to start a Kafka broker and share its address between the Quarkus application, the Camel component, and the Citrus test endpoints.

Citrus solves this with a single annotation: `@KafkaContainerSupport`. It starts a real Kafka broker via Testcontainers and automatically publishes the bootstrap server address so that all three frameworks — Quarkus, Camel, and Citrus — connect to the same broker instance. No manual property overrides, no test-specific configuration files.

# Provisioning the Kafka broker

The test class starts with three annotations that set up the complete test infrastructure:

```java
@QuarkusTest
@CitrusSupport
@KafkaContainerSupport(port = 9092, version = "4.2.0",
        containerLifecycleListener = QuarkusApplicationTest.KafkaConfigurer.class)
class QuarkusApplicationTest implements TestActionSupport {
    // ...
}
```

`@QuarkusTest` starts the Quarkus application in test mode and activates the Camel routes. `@CitrusSupport` layers the Citrus framework on top of the Quarkus lifecycle. And `@KafkaContainerSupport` is where the magic happens — it launches a Kafka 4.2.0 container on port 9092 and wires everything together.

The `containerLifecycleListener` parameter points to a custom listener that runs after the Kafka container starts. This is where you perform any broker-level setup your tests need, such as creating topics:

```java
public static class KafkaConfigurer implements ContainerLifecycleListener<KafkaContainer> {
    @Override
    public Map<String, String> started(KafkaContainer container) {
        try (Admin adminClient = Admin.create(
                Collections.singletonMap(
                    AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG,
                    container.getBootstrapServers()))) {

            CreateTopicsResult result = adminClient.createTopics(Set.of(
                new NewTopic("words-in", 1, (short) 1),
                new NewTopic("words-out", 1, (short) 1)
            ));

            result.all().get();
        }

        return Collections.emptyMap();
    }
}
```

The listener receives the running `KafkaContainer` instance, creates an admin client pointing to the container's bootstrap servers, and creates the two topics the Camel route needs. The topics are ready before the Quarkus application and the Citrus endpoints start connecting.

The `ContainerLifecycleListener` is flexible. Beyond topic creation you could use it to configure ACLs, set up consumer groups, or seed test data. The `started()` method can also return a map of additional system properties that get injected into the test context.

# Configuring Kafka endpoints

With the broker running and topics created, you need to tell Citrus where to send and receive messages. Citrus uses a declarative annotation-based approach:

```java
@CitrusEndpoint
@KafkaEndpointConfig(topic = "words-in")
KafkaEndpoint wordsIn;

@CitrusEndpoint
@KafkaEndpointConfig(topic = "words-out")
KafkaEndpoint wordsOut;
```

Each field annotated with `@CitrusEndpoint` becomes a fully configured Kafka endpoint. The `@KafkaEndpointConfig` annotation specifies the topic name. The bootstrap server address is automatically resolved from the `@KafkaContainerSupport` configuration — you do not need to specify it explicitly.

Two endpoints are defined: `wordsIn` for sending messages into the Camel route's input topic, and `wordsOut` for receiving and validating the transformed output. Citrus handles Kafka producer and consumer creation, serialization, and connection management behind the scenes.

# Writing the test

With the infrastructure and endpoints in place, the actual test is remarkably concise:

```java
@CitrusResource
GherkinTestActionRunner runner;

@Test
void shouldHandleEvents() {
    runner.when(
        send()
            .endpoint(wordsIn)
            .message()
            .body("Howdy")
    );

    runner.then(
        receive()
            .endpoint(wordsOut)
            .message()
            .body(">> HOWDY")
    );
}
```

The `GherkinTestActionRunner` provides a fluent API with Given-When-Then semantics. The **when** block sends the string `"Howdy"` to the `words-in` topic. The Camel route picks it up, transforms it, and publishes the result to `words-out`. The **then** block verifies that a message with body `">> HOWDY"` arrives on the output topic.

That single assertion covers the entire processing pipeline: Kafka consumption by Camel, the uppercase transformation, the prefix formatting, and the Kafka production on the output side. If the received message does not match the expected body, Citrus fails the test with a clear diff showing exactly what went wrong.

While the Camel route processes messages asynchronously, the Citrus test runs synchronously. The `receive()` action blocks until a message arrives on the output topic or a timeout occurs. This approach ensures deterministic test execution without flaky sleeps or polling loops.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events that unfolds behind the scenes:

1. **`@KafkaContainerSupport` launches** a Kafka 4.2.0 container on port 9092 via Testcontainers.
2. **The lifecycle listener** creates the `words-in` and `words-out` topics using the Kafka Admin API.
3. **Quarkus starts** the application in test mode with the bootstrap servers pointing to the test Kafka broker.
4. **Apache Camel discovers** the `Routes` class and starts the Kafka consumer on `words-in`.
5. **Citrus initializes** the `wordsIn` and `wordsOut` endpoints connected to the same broker.
6. **The test sends** `"Howdy"` to the `words-in` topic through the `wordsIn` endpoint.
7. **The Camel route consumes** the message, transforms it to `">> HOWDY"`, and produces it to `words-out`.
8. **Citrus receives** the message from the `words-out` topic and validates it against the expected body.
9. **The test completes** and the Kafka container is stopped and cleaned up automatically.

The critical point is step 5: all three frameworks — Quarkus, Apache Camel, and Citrus — share the same Kafka broker instance. The `@KafkaContainerSupport` annotation takes care of distributing the bootstrap server configuration so you do not have to wire it manually.

There is one small piece of test configuration worth mentioning. Because Citrus and Quarkus share the same classpath, you need to tell Quarkus CDI to allow split packages from the Citrus framework. This is done in `src/test/resources/application.properties`:

```properties
quarkus.arc.ignored-split-packages=org.citrusframework.*
```

This avoids CDI bean discovery conflicts and is a one-line addition that you set once and forget.

# The complete test class

Here is the full test in one place for easy reference:

```java
@QuarkusTest
@CitrusSupport
@KafkaContainerSupport(port = 9092, version = "4.2.0",
        containerLifecycleListener = QuarkusApplicationTest.KafkaConfigurer.class)
class QuarkusApplicationTest implements TestActionSupport {

    @CitrusEndpoint
    @KafkaEndpointConfig(topic = "words-in")
    KafkaEndpoint wordsIn;

    @CitrusEndpoint
    @KafkaEndpointConfig(topic = "words-out")
    KafkaEndpoint wordsOut;

    @CitrusResource
    GherkinTestActionRunner runner;

    @Test
    void shouldHandleEvents() {
        runner.when(
            send()
                .endpoint(wordsIn)
                .message()
                .body("Howdy")
        );

        runner.then(
            receive()
                .endpoint(wordsOut)
                .message()
                .body(">> HOWDY")
        );
    }

    public static class KafkaConfigurer
            implements ContainerLifecycleListener<KafkaContainer> {
        @Override
        public Map<String, String> started(KafkaContainer container) {
            try (Admin adminClient = Admin.create(
                    Collections.singletonMap(
                        AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG,
                        container.getBootstrapServers()))) {

                adminClient.createTopics(Set.of(
                    new NewTopic("words-in", 1, (short) 1),
                    new NewTopic("words-out", 1, (short) 1)
                )).all().get();
            }

            return Collections.emptyMap();
        }
    }
}
```

The complete test weighs in at around 50 lines of code. It provisions a real Kafka broker, creates topics, starts the Quarkus application with Apache Camel, sends a message through the entire pipeline, and validates the output. That is the power of Citrus combined with Camel and Quarkus.

# Where to go from here

This example covers the fundamentals, but Citrus has much more to offer for Apache Camel and Kafka testing scenarios:

- **Message headers and keys**: Kafka records carry headers and message keys alongside the payload. Citrus lets you set and verify both in your send and receive actions, which is essential for testing content-based routing in Camel.
- **JSON validation**: Add the `citrus-validation-json` module and Citrus will compare received JSON payloads field by field, supporting ignore patterns, validation matchers, and flexible element ordering.
- **Complex Camel patterns**: Test content-based routing, message filters, aggregation, and error handling. Citrus can simulate multiple input and output topics to cover multi-step integration scenarios.
- **Error scenarios**: Test what happens when the application receives malformed input or when downstream services are unavailable. Citrus gives you full control over the messages you send, making negative testing straightforward.
- **Dev services comparison**: If your Quarkus application uses SmallRye Reactive Messaging instead of Camel's native Kafka component, Quarkus dev services can handle broker provisioning automatically. The [event-driven Kafka sample](/samples/quarkus-event-driven-kafka/) on this site demonstrates that approach. Choose `@KafkaContainerSupport` when you need explicit control over the broker or when dev services do not apply.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers Kafka endpoints, Quarkus integration, Apache Camel support, and the many other messaging transports that Citrus supports.
