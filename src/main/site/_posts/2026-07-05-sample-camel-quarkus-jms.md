---
layout: sample
title: Apache Camel Quarkus - JMS Integration Testing
name: camel-jms
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel JMS routes in Quarkus with Citrus and ActiveMQ Artemis
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-jms/
---

Apache Camel is a natural fit for JMS-based integration in Quarkus. Where a pure Quarkus approach requires multiple classes to bridge JMS queues with reactive channels, a single Camel route can express the same flow in a few lines of code.

But testing that flow end to end is another story. You need a real JMS broker, a shared connection factory, and a test framework that can send and receive messages across queues while the application processes them asynchronously. Getting all of that to work together without fragile test infrastructure takes some careful wiring.

This post walks you through testing an Apache Camel JMS route in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus shares the JMS connection factory with both Quarkus and Camel, sends messages into the route, and validates the output with clean, readable test code. By the end you will have a clear recipe for testing your own Camel JMS routes in Quarkus.

# The application under test

The example application is intentionally simple so we can focus on the testing side. It is a Quarkus service that uses an Apache Camel route to consume text messages from a JMS queue, transform them to uppercase, and publish the result to an output queue.

```
JMS Queue (words-in) --> Apache Camel Route (Transform) --> JMS Queue (words-out)
```

The entire integration logic lives in a single Camel `RouteBuilder`:

```java
public class Routes extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("jms:words-in")
            .setBody(exchange -> ">> " + exchange.getIn().getBody().toString().toUpperCase())
            .to("jms:words-out");
    }
}
```

Three lines of fluent API code handle the complete integration flow. The route consumes messages from the `words-in` JMS queue, transforms each message body to uppercase with a `">> "` prefix, and publishes the result to the `words-out` queue.

Compare this with the multi-class approach you would need without Camel: a JMS consumer class, a reactive messaging processor, and a JMS producer class. Camel collapses all of that into a single, readable route definition. Quarkus automatically discovers any class extending `RouteBuilder` on the classpath and starts the route when the application boots.

The `jms:` component prefix tells Camel to use the JMS component, which Quarkus auto-configures to use the Artemis connection factory. No additional configuration code is needed — just the broker address in `application.properties`:

```properties
quarkus.artemis.url=tcp://localhost:61616
quarkus.artemis.username=quarkus
quarkus.artemis.password=quarkus
```

That is the entire application. Now let's verify it.

# Adding Citrus to the project

Citrus is highly modular, so you only pull in the capabilities you actually need. For this Camel JMS scenario you add three test-scoped dependencies to the Maven POM:

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

# Setting up the test class

The test class starts with three annotations that set up the complete test infrastructure:

```java
@QuarkusTest
@CitrusSupport
@WithTestResource(ArtemisTestResource.class)
class QuarkusApplicationTest implements TestActionSupport {
    // ...
}
```

`@QuarkusTest` starts the Quarkus application in test mode and activates the Camel routes. `@WithTestResource(ArtemisTestResource.class)` provisions an ActiveMQ Artemis broker via Testcontainers and configures the Quarkus connection factory to point to it. `@CitrusSupport` layers the Citrus framework on top of that lifecycle so you can inject Citrus-specific resources into the test class.

The `TestActionSupport` interface is a convenience that gives you static access to test actions like `send()` and `receive()` without extra imports.

# Sharing the connection factory across three frameworks

This is the most important piece of the setup. When your application uses Apache Camel's JMS component, three frameworks need to talk to the same broker: Quarkus manages the connection factory, Camel uses it for its `jms:` component, and Citrus needs it for its test endpoints. All three must share the same `ConnectionFactory` instance.

Citrus solves this with a single pair of annotations:

```java
@Inject
@BindToRegistry
ConnectionFactory connectionFactory;
```

`@Inject` tells Quarkus to provide the connection factory it configured for the Artemis test resource. `@BindToRegistry` is a Citrus annotation that registers this same instance in the Citrus component registry. From that point on, every Citrus JMS endpoint automatically picks up this connection factory.

The result is that all three frameworks — Quarkus, Apache Camel, and Citrus — communicate through the same broker connection without any manual wiring. This two-annotation pattern is the key to seamless JMS testing with Camel in Quarkus.

# Configuring JMS endpoints

With the broker running and the connection factory shared, you need to tell Citrus which JMS queues to use. Citrus offers a declarative annotation-based approach:

```java
@CitrusEndpoint
@JmsEndpointConfig(destinationName = "words-in")
JmsEndpoint wordsIn;

@CitrusEndpoint
@JmsEndpointConfig(destinationName = "words-out")
JmsEndpoint wordsOut;
```

Each field annotated with `@CitrusEndpoint` becomes a fully configured JMS endpoint. The `@JmsEndpointConfig` annotation specifies the JMS destination (queue) name. The connection factory is automatically resolved from the Citrus registry — the one you registered with `@BindToRegistry` in the previous step. No explicit wiring needed.

Two endpoints are defined: `wordsIn` for sending messages into the Camel route's input queue, and `wordsOut` for receiving and validating the transformed output.

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

The `GherkinTestActionRunner` provides a fluent API with Given-When-Then semantics. The **when** block sends the string `"Howdy"` to the `words-in` queue. The Camel route picks it up, transforms it to uppercase with the `">> "` prefix, and publishes the result to `words-out`. The **then** block verifies that a message with body `">> HOWDY"` arrives on the output queue.

That single assertion covers the entire processing pipeline: JMS consumption by Camel, the uppercase transformation, the prefix formatting, and the JMS production on the output side. If the received message does not match the expected body, Citrus fails the test with a clear diff showing exactly what went wrong.

While the Camel route processes messages asynchronously, the Citrus test runs synchronously. The `receive()` action blocks until a message arrives on the output queue or a timeout occurs. This approach ensures deterministic test execution without flaky sleeps or polling loops.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events that unfolds behind the scenes:

1. **The Artemis test resource** launches an ActiveMQ Artemis container via Testcontainers and configures the connection factory.
2. **Quarkus starts** the application in test mode with the connection factory pointing to the test broker.
3. **Apache Camel discovers** the `Routes` class and starts the JMS consumer on the `words-in` queue.
4. **Citrus registers** the connection factory and initializes the `wordsIn` and `wordsOut` JMS endpoints.
5. **The test sends** `"Howdy"` to the `words-in` queue through the `wordsIn` endpoint.
6. **The Camel route consumes** the message, transforms it to `">> HOWDY"`, and produces it to `words-out`.
7. **Citrus receives** the message from the `words-out` queue and validates it against the expected body.
8. **The test completes** and the Artemis container is stopped and cleaned up automatically.

The critical point is step 4: all three frameworks — Quarkus, Apache Camel, and Citrus — share the same connection factory and broker instance. The `@BindToRegistry` annotation takes care of distributing the connection factory so you do not have to wire it manually.

There is one small piece of test configuration worth mentioning. Because Citrus and Quarkus share the same classpath, you need to tell Quarkus CDI to allow split packages from the Citrus framework. This is done in `src/test/resources/application.properties`:

```properties
quarkus.arc.ignored-split-packages=org.citrusframework.*
```

The test configuration also clears the broker credentials because the Artemis test resource starts an unauthenticated broker by default:

```properties
%test.quarkus.artemis.username=
%test.quarkus.artemis.password=
```

Both are one-time additions that you set once and forget.

# The complete test class

Here is the full test in one place for easy reference:

```java
@QuarkusTest
@CitrusSupport
@WithTestResource(ArtemisTestResource.class)
class QuarkusApplicationTest implements TestActionSupport {

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

Under 40 lines of code for a complete integration test that provisions a real JMS broker, starts a Quarkus application with Apache Camel, sends a message through the entire pipeline, and validates the output. That is the power of Citrus combined with Camel and Quarkus.

# Camel JMS vs plain Quarkus JMS

If you have read the [event-driven JMS sample](/samples/quarkus-event-driven-jms/) on this site, you will notice that the Citrus test structure is almost identical. The connection factory sharing, endpoint configuration, and test actions are the same. The difference is in the application code.

The plain Quarkus approach uses three separate classes — a JMS consumer, a reactive messaging processor, and a JMS producer — to bridge JMS queues with SmallRye Reactive Messaging channels. The Apache Camel approach replaces all of that with a single three-line route. Camel handles the JMS consumption, transformation, and production natively, removing the need for manual message bridging.

From a testing perspective, this means the same Citrus patterns work regardless of whether your Quarkus application uses Camel or reactive messaging for JMS integration. Citrus abstracts the messaging transport behind its endpoint model, so your test code stays consistent across different application architectures.

# Where to go from here

This example covers the fundamentals, but Citrus has much more to offer for Apache Camel JMS testing scenarios:

- **Message headers and properties**: JMS messages carry headers and custom properties alongside the payload. Citrus lets you set and verify both in your send and receive actions, which is useful for testing header-based routing in Camel.
- **JSON validation**: Add the `citrus-validation-json` module and Citrus will compare received JSON payloads field by field, supporting ignore patterns, validation matchers, and flexible element ordering.
- **Complex Camel patterns**: Test content-based routing, message filters, aggregation, and error handling. Citrus can simulate multiple input and output queues to cover multi-step integration scenarios.
- **Error scenarios**: Test what happens when the application receives malformed input or when downstream services are unavailable. Citrus gives you full control over the messages you send, making negative testing straightforward.
- **Kafka comparison**: If your Camel route uses Kafka instead of JMS, the [Camel Kafka sample](/samples/camel-quarkus-kafka/) on this site demonstrates the equivalent setup with `@KafkaContainerSupport` for broker provisioning. The test actions stay the same — only the endpoint configuration changes.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers JMS endpoints, Quarkus integration, Apache Camel support, and the many other messaging transports that Citrus supports.
