---
layout: sample
title: Quarkus - Event-Driven JMS
name: event-driven-jms
image: /img/icons/quarkus.png
folder: common
group: quarkus
description: How Citrus helps to verify event driven JMS messaging in a Quarkus application
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/quarkus-event-driven-jms/
---

JMS is still one of the most widely used messaging standards in enterprise applications. ActiveMQ Artemis brings a modern, high-performance JMS broker to the table, and Quarkus makes it easy to build reactive services on top of it. 

But testing the asynchronous message flows in these applications is often where things get tricky. How do you send a message to a JMS queue, wait for the application to process it, and then verify the output on another queue — all in a deterministic, repeatable test?

This post walks you through testing an event-driven Quarkus application with the [Citrus](https://citrusframework.org) integration testing framework and JMS. You will see how Citrus connects to JMS queues, shares the broker connection with Quarkus, and validates messages with just a few lines of code. By the end you will have a clear recipe for adding JMS integration tests to your own Quarkus project.

# The application under test

The example application is intentionally simple so we can focus on the testing side. It is a Quarkus service that consumes text messages from a JMS queue, transforms them to uppercase, and publishes the result to an output queue.

```
JMS Queue (words-in) --> Transform to Uppercase --> JMS Queue (words-out)
```

The implementation uses three components that work together to bridge JMS with SmallRye Reactive Messaging channels.

The `WordConsumer` listens on the `words-in` JMS queue using the standard JMS API and emits each message into a reactive channel:

```java
@ApplicationScoped
public class WordConsumer implements Runnable {

    @Channel("words-in")
    Emitter<String> emitter;

    @Inject
    ConnectionFactory connectionFactory;

    @Override
    public void run() {
        try (JMSContext context = connectionFactory.createContext(JMSContext.AUTO_ACKNOWLEDGE)) {
            JMSConsumer consumer = context.createConsumer(context.createQueue("words-in"));
            while (true) {
                Message message = consumer.receive();
                if (message == null) return;
                emitter.send(message.getBody(String.class));
            }
        }
    }
}
```

The `EventDrivenApplication` picks up messages from the reactive channel, transforms them, and hands the result to a producer:

```java
@ApplicationScoped
public class EventDrivenApplication {

    @Inject
    WordProducer producer;

    @Incoming("words-in")
    @Outgoing("uppercase")
    public String toUpperCase(String message) {
        return message.toUpperCase();
    }

    @Incoming("uppercase")
    public void sink(String word) {
        producer.send(">> " + word);
    }
}
```

Finally, the `WordProducer` sends the processed message to the `words-out` JMS queue:

```java
@ApplicationScoped
public class WordProducer {

    @Inject
    ConnectionFactory connectionFactory;

    public void send(String word) {
        try (JMSContext context = connectionFactory.createContext(JMSContext.AUTO_ACKNOWLEDGE)) {
            context.createProducer().send(context.createQueue("words-out"), word);
        }
    }
}
```

The flow starts and ends with JMS queues, but uses reactive messaging channels internally. This is a common pattern in Quarkus applications that integrate with legacy or enterprise messaging systems. Now let's verify it.

# Adding Citrus to the project

Citrus is highly modular, so you only pull in the capabilities you actually need. For this JMS scenario you add three test-scoped dependencies to the Maven POM:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-jms</artifactId>
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
- **citrus-jms** provides JMS endpoint implementations with built-in producer and consumer capabilities and message validation.
- **citrus-junit-jupiter** connects Citrus to JUnit 5, the test engine that Quarkus uses under the hood.

You also need the Quarkus Artemis test resource to provision a real broker automatically:

```xml
<dependency>
    <groupId>io.quarkiverse.artemis</groupId>
    <artifactId>quarkus-test-artemis</artifactId>
    <scope>test</scope>
</dependency>
```

This dependency starts an ActiveMQ Artemis container via Testcontainers when the tests run. No manual broker setup, no Docker Compose files, no cleanup scripts.

# Enabling Citrus in a Quarkus test

The bridge between Quarkus and Citrus is a single annotation. Add `@CitrusSupport` next to `@QuarkusTest` and the framework takes care of the rest:

```java
@QuarkusTest
@CitrusSupport
@WithTestResource(ArtemisTestResource.class)
class EventDrivenApplicationTest implements TestActionSupport {

    @CitrusResource
    GherkinTestActionRunner runner;

    // endpoint configurations and test methods
}
```

A few things are worth noting here.

`@QuarkusTest` starts the application in test mode. `@WithTestResource(ArtemisTestResource.class)` provisions the Artemis broker and configures the Quarkus connection factory to point to it. `@CitrusSupport` layers Citrus on top of that lifecycle so you can inject Citrus-specific resources into the test class.

The `GherkinTestActionRunner` is the entry point for all Citrus test actions. It provides a fluent API with Given-When-Then semantics that makes tests read almost like specifications. You inject it with the `@CitrusResource` annotation.

The `TestActionSupport` interface is a convenience that gives you static access to test actions like `send()` and `receive()` without extra imports. It keeps the test code clean and focused on the intent.

# Sharing the connection factory

With Kafka, Citrus and Quarkus connect to the same broker by sharing a property like `kafka.bootstrap.servers`. JMS works differently — it uses a `ConnectionFactory` object. The challenge is to make both the application and the test use the exact same factory instance pointing to the same Artemis broker.

Citrus solves this with a single pair of annotations:

```java
@Inject
@BindToRegistry
ConnectionFactory connectionFactory;
```

`@Inject` tells Quarkus to provide the connection factory it configured for the Artemis test resource. `@BindToRegistry` is a Citrus annotation that registers this same instance in the Citrus component registry. From that point on, every Citrus JMS endpoint automatically picks up this connection factory.

This two-annotation pattern is the key to seamless JMS testing with Quarkus. Both sides share the same broker connection, and there is nothing else to configure.

# Configuring JMS endpoints

Before you can send and receive messages in a test, you need to tell Citrus which JMS queues to use. Citrus offers a declarative annotation-based approach:

```java
@CitrusEndpoint
@JmsEndpointConfig(destinationName = "words-in")
JmsEndpoint wordsIn;

@CitrusEndpoint
@JmsEndpointConfig(destinationName = "words-out")
JmsEndpoint wordsOut;
```

Each field annotated with `@CitrusEndpoint` becomes a fully configured JMS endpoint. The `@JmsEndpointConfig` annotation specifies the JMS destination (queue) name.

The connection factory is automatically resolved from the Citrus registry — the one you registered with `@BindToRegistry` in the previous step. No explicit wiring needed. Two endpoints are defined: one for sending messages to the application's input queue (`wordsIn`) and one for receiving from the output queue (`wordsOut`).

# Writing the test

With the endpoints in place, the actual test is remarkably concise:

```java
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

The test sends the string `"Howdy"` to the `words-in` queue and then verifies that the `words-out` queue receives `">> HOWDY"`. That single assertion covers the entire processing pipeline: JMS consumption, the reactive channel handoff, the uppercase transformation, the prefix formatting, and the JMS production on the output side.

The Gherkin-style `when` and `then` methods clearly communicate intent. The **when** block describes the stimulus, the action that triggers the application behavior. The **then** block describes the expected outcome. If the received message does not match the expected body, Citrus fails the test with a clear diff showing exactly what went wrong.

While the application processes messages asynchronously across multiple components, the Citrus test runs synchronously. The `send()` action publishes a message to the JMS queue, the application asynchronously consumes, transforms, and produces the message, and the `receive()` action blocks until a message arrives on the output queue or a timeout occurs. This approach ensures deterministic test execution without flaky sleeps or polling loops.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events that unfolds behind the scenes:

1. **Quarkus starts** the application in test mode.
2. **The Artemis test resource** launches an ActiveMQ Artemis container via Testcontainers and configures the connection factory.
3. **Citrus registers** the connection factory and initializes the `wordsIn` and `wordsOut` JMS endpoints.
4. **The test sends** `"Howdy"` to the `words-in` queue through the `wordsIn` endpoint.
5. **The application consumes** the message, transforms it to `"HOWDY"`, prefixes it with `">> "`, and publishes `">> HOWDY"` to the `words-out` queue.
6. **Citrus receives** the message from the `words-out` queue through the `wordsOut` endpoint and validates it against the expected body.
7. **The test completes** and the Artemis container is stopped automatically.

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
@WithTestResource(ArtemisTestResource.class)
class EventDrivenApplicationTest implements TestActionSupport {

    @Inject
    @BindToRegistry
    ConnectionFactory connectionFactory;

    @CitrusEndpoint
    @JmsEndpointConfig(destinationName = "words-in")
    JmsEndpoint wordsIn;

    @CitrusEndpoint
    @JmsEndpointConfig(destinationName = "words-out")
    JmsEndpoint wordsOut;

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
}
```

Under 50 lines of code for a complete integration test that sends a message to a JMS queue, waits for the application to process it, receives the result from a different queue, and validates the output. That is the power of Citrus combined with Quarkus test resources.

# JMS vs Kafka — same pattern, different transport

If you have read the [Kafka event-driven sample](/samples/quarkus-event-driven-kafka/) on this site, the structure of this test will look very familiar. And that is by design. Citrus abstracts the messaging transport behind its endpoint model, so switching from Kafka to JMS is mostly a matter of swapping endpoint types and adjusting how the broker connection is shared.

With Kafka, Citrus picks up the `kafka.bootstrap.servers` property that Quarkus dev services publishes. With JMS, you share the `ConnectionFactory` object through `@BindToRegistry`. The test actions — `send()`, `receive()`, the Gherkin-style runner — stay exactly the same regardless of which transport you use.

This consistency means your team can apply the same testing patterns across different messaging technologies without learning a new API for each one.

# Where to go from here

This example covers the fundamentals, but Citrus has much more to offer for JMS testing scenarios:

- **Message headers and properties**: JMS messages carry headers and custom properties alongside the payload. Citrus lets you set and verify both in your send and receive actions.
- **JSON validation**: Add the `citrus-validation-json` module and Citrus will compare received JSON payloads field by field, supporting ignore patterns, validation matchers, and flexible element ordering.
- **Multiple queues and complex flows**: Real applications often consume from several queues and produce to several others. You can define as many endpoints as you need and orchestrate multi-step test scenarios.
- **Error scenarios**: Test what happens when the application receives malformed input or when downstream services are unavailable. Citrus gives you full control over the messages you send, making negative testing straightforward.
- **Endpoint configuration classes**: When your test suite grows, you can extract endpoint definitions into shared `@CitrusConfiguration` classes and reuse them across multiple tests. The [Quarkus sample](/samples/quarkus/) on this site demonstrates this pattern in detail.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers JMS endpoints, Quarkus integration, and the many other messaging transports that Citrus supports.
