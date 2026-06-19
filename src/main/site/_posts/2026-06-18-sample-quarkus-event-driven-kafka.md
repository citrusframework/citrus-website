---
layout: sample
title: Quarkus - Event-Driven Kafka
name: event-driven-kafka
image: /img/icons/quarkus.png
folder: common
group: quarkus
description: How Citrus helps to verify event driven Kafka messaging in a Quarkus application
categories: [samples]
permalink: /samples/quarkus-event-driven-kafka/
---

Event-driven architectures powered by Apache Kafka are everywhere. Microservices publish and consume events asynchronously, and the reactive nature of these systems makes them fast and scalable. But it also makes them harder to test. How do you verify that a message entering one Kafka topic produces the correct output on another? How do you write tests that are deterministic, readable, and run against a real broker instead of fragile mocks?

This post walks you through testing an event-driven Quarkus application with the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus connects to Kafka topics, sends and receives messages, and validates the results with just a few lines of code. By the end you will have a clear recipe for adding Citrus to your own Quarkus project.

# The application under test

The example application is intentionally simple so we can focus on the testing side. It is a Quarkus service that listens for incoming text messages on a Kafka topic, transforms them to uppercase, and publishes the result to an output topic.

```
Kafka Topic (words-in) --> Transform to Uppercase --> Kafka Topic (words-out)
```

The implementation uses SmallRye Reactive Messaging, the MicroProfile standard that Quarkus adopts for event-driven communication. Two methods handle the entire flow:

```java
@ApplicationScoped
public class EventDrivenApplication {

    @Channel("words-out")
    Emitter<String> emitter;

    @Incoming("words-in")
    @Outgoing("uppercase")
    public String toUpperCase(String message) {
        return message.toUpperCase();
    }

    @Incoming("uppercase")
    public void sink(String word) {
        emitter.send(">> " + word);
    }
}
```

The `toUpperCase` method receives each message from the `words-in` channel and pushes the uppercased result to an internal channel called `uppercase`. The `sink` method picks it up, prepends `">> "`, and emits the final string to the `words-out` channel. The channel-to-topic mapping lives in `application.properties`:

```properties
mp.messaging.incoming.words-in.auto.offset.reset=earliest
mp.messaging.incoming.words-in.topic=words-in
mp.messaging.outgoing.words-out.topic=words-out
```

That is the entire production code. Now let's verify it.

# Adding Citrus to the project

Citrus is highly modular, so you only pull in the capabilities you actually need. For this Kafka scenario you add three test-scoped dependencies to the Maven POM:

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
    <artifactId>citrus-junit-jupiter</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Here is what each module brings to the table:

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle. It hooks into `@QuarkusTest` so that Citrus endpoints and test runners are available without any extra configuration.
- **citrus-kafka** provides Kafka endpoint implementations with built-in producer and consumer capabilities, message serialization, and topic configuration.
- **citrus-junit-jupiter** connects Citrus to JUnit 5, the test engine that Quarkus uses under the hood.

No additional infrastructure setup is required. Quarkus dev services will automatically start a Kafka broker via Testcontainers when the tests run. Citrus connects to that same broker, which means your tests hit a real Kafka instance, not a mock.

# Enabling Citrus in a Quarkus test

The bridge between Quarkus and Citrus is a single annotation. Add `@CitrusSupport` next to `@QuarkusTest` and the framework takes care of the rest:

```java
@QuarkusTest
@CitrusSupport
class EventDrivenApplicationTest implements TestActionSupport {

    @CitrusResource
    GherkinTestActionRunner runner;

    // endpoint configurations and test methods
}
```

A few things are worth noting here.

`@QuarkusTest` starts the application in test mode and activates dev services. This is where the Kafka Testcontainer gets launched. `@CitrusSupport` layers Citrus on top of that lifecycle so you can inject Citrus-specific resources into the test class.

The `GherkinTestActionRunner` is the entry point for all Citrus test actions. It provides a fluent API with Given-When-Then semantics that makes tests read almost like specifications. You inject it with the `@CitrusResource` annotation.

The `TestActionSupport` interface is a convenience that gives you static access to test actions like `send()` and `receive()` without extra imports. It keeps the test code clean and focused on the intent.

# Configuring Kafka endpoints

Before you can send and receive messages in a test, you need to tell Citrus which Kafka topics to use and how to connect. Citrus offers a declarative annotation-based approach that fits naturally into the test class:

```java
@CitrusEndpoint
@KafkaEndpointConfig(topic = "words-in",
        server = "${kafka.bootstrap.servers}")
KafkaEndpoint wordsIn;

@CitrusEndpoint
@KafkaEndpointConfig(topic = "words-out",
        server = "${kafka.bootstrap.servers}")
KafkaEndpoint wordsOut;
```

Each field annotated with `@CitrusEndpoint` becomes a fully configured Kafka endpoint. The `@KafkaEndpointConfig` annotation specifies the topic name and the broker address.

The `${kafka.bootstrap.servers}` property is the key to the seamless integration with Quarkus dev services. When Quarkus starts the Kafka Testcontainer, it publishes the broker address under this property. Citrus picks it up automatically, so both the application and the test connect to the exact same Kafka instance. No hardcoded ports, no environment-specific configuration files, no manual container management.

# Writing the test

With the endpoints in place, the actual test is remarkably concise:

```java
@Test
void shouldHandleEvents() {
    runner.when(
        send()
            .endpoint(wordsIn)
            .message()
            .body("Hi")
    );

    runner.then(
        receive()
            .endpoint(wordsOut)
            .message()
            .body(">> HI")
    );
}
```

The test sends the string `"Hi"` to the `words-in` topic and then verifies that the `words-out` topic receives `">> HI"`. That single assertion covers the entire processing pipeline: Kafka consumption, the uppercase transformation, the prefix formatting, and the Kafka production on the output side.

The Gherkin-style `when` and `then` methods are not just syntactic sugar. They clearly communicate intent. The **when** block describes the stimulus, the action that triggers the application behavior. The **then** block describes the expected outcome. If the received message does not match the expected body, Citrus fails the test with a clear diff showing exactly what went wrong.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events that unfolds behind the scenes:

1. **Quarkus starts** the application in test mode and activates dev services.
2. **Dev services launches** a Kafka container via Testcontainers and publishes the broker address as `kafka.bootstrap.servers`.
3. **Citrus initializes** the `wordsIn` and `wordsOut` endpoints using the broker address from the property.
4. **The test sends** `"Hi"` to the `words-in` topic through the `wordsIn` endpoint.
5. **The application consumes** the message, transforms it to `"HI"`, prefixes it with `">> "`, and publishes `">> HI"` to the `words-out` topic.
6. **Citrus receives** the message from the `words-out` topic through the `wordsOut` endpoint and validates it against the expected body.
7. **The test completes** and Quarkus tears down the Kafka container automatically.

No manual Kafka setup, no Docker Compose files to maintain, no cleanup scripts. The entire infrastructure lifecycle is managed for you.

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
class EventDrivenApplicationTest implements TestActionSupport {

    @CitrusEndpoint
    @KafkaEndpointConfig(topic = "words-in",
            server = "${kafka.bootstrap.servers}")
    KafkaEndpoint wordsIn;

    @CitrusEndpoint
    @KafkaEndpointConfig(topic = "words-out",
            server = "${kafka.bootstrap.servers}")
    KafkaEndpoint wordsOut;

    @CitrusResource
    GherkinTestActionRunner runner;

    @Test
    void shouldHandleEvents() {
        runner.when(
            send()
                .endpoint(wordsIn)
                .message()
                .body("Hi")
        );

        runner.then(
            receive()
                .endpoint(wordsOut)
                .message()
                .body(">> HI")
        );
    }
}
```

Under 50 lines of code for a complete integration test that sends a message to Kafka, waits for the application to process it, receives the result from a different topic, and validates the output. That is the power of Citrus combined with Quarkus dev services.

# Where to go from here

This example covers the fundamentals, but Citrus has much more to offer for Kafka testing scenarios:

- **Message headers**: Kafka records carry headers and message keys alongside the payload. Citrus lets you set and verify both in your send and receive actions.
- **JSON validation**: Add the `citrus-validation-json` module and Citrus will compare received JSON payloads field by field, supporting ignore patterns, validation matchers, and flexible element ordering.
- **Multiple topics and complex flows**: Real applications often consume from several topics and produce to several others. You can define as many endpoints as you need and orchestrate multi-step test scenarios.
- **Error scenarios**: Test what happens when the application receives malformed input or when downstream services are unavailable. Citrus gives you full control over the messages you send, making negative testing straightforward.
- **Endpoint configuration classes**: When your test suite grows, you can extract endpoint definitions into shared `@CitrusConfiguration` classes and reuse them across multiple tests. The [Quarkus sample](/samples/quarkus/) on this site demonstrates this pattern in detail.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers Kafka endpoints, Quarkus integration, and the many other messaging transports that Citrus supports.
