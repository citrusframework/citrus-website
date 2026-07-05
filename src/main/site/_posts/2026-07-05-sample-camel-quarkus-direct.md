---
layout: sample
title: Apache Camel Quarkus - Direct Endpoint Testing
name: camel-direct
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel routes with direct and mock endpoints in Quarkus using Citrus
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-direct/
---

When you are developing Apache Camel routes, the first thing you want to verify is whether the route logic itself works correctly — does the transformation produce the right output? Does the routing condition evaluate as expected? These are questions you can answer with just connecting to the Camel route and verifying the route outcome with a mocked endpoint.

Camel's `direct:` and `mock:` components make this possible. A `direct:` endpoint accepts messages synchronously in memory, and a `mock:` endpoint captures everything sent to it for later assertion. Together they let you test route logic in complete isolation — no containers, no network, no waiting.

This post walks you through testing an Apache Camel route in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework with direct and mock endpoints. You will see how Citrus shares the CamelContext with Quarkus, sends messages into routes via the `camel:` URI scheme, and combines its own test DSL with Camel's MockEndpoint assertions. 

By the end you will have a recipe for fast, isolated Camel route testing that runs in under two seconds with zero external infrastructure.

# The application under test

The example application is as simple as it gets. A single Camel route consumes messages from a `direct:` endpoint, transforms the body to uppercase with a prefix, and sends the result to a `mock:` endpoint.

```
Direct Endpoint (direct:words-in) --> Transform --> Mock Endpoint (mock:words-out)
```

```java
public class Routes extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("direct:words-in")
            .setBody(exchange -> ">> " + exchange.getIn().getBody().toString().toUpperCase())
            .to("mock:words-out");
    }
}
```

The `direct:` component provides synchronous, in-memory message passing within a single CamelContext. There is no network transport, no serialization, and no external dependency. Messages are passed directly from the caller to the route in the same thread.

The `mock:` component is Camel's built-in test endpoint. It captures every message sent to it and provides a rich assertion API — you can verify message count, body content, headers, and ordering. While `mock:` is designed primarily for testing, it can also serve as a temporary placeholder during early development before the real endpoint is ready.

This route is intentionally minimal so we can focus on the testing patterns. The same approach scales to complex routes with content-based routing, splitting, aggregation, or error handling — as long as the entry point is a `direct:` endpoint, Citrus can send messages into it.

# Adding Citrus to the project

Since this example uses no external transports, the dependency list is the smallest of any sample on this site:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-camel</artifactId>
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

You also need Camel's Quarkus test support for mock endpoint access:

```xml
<dependency>
    <groupId>org.apache.camel.quarkus</groupId>
    <artifactId>camel-quarkus-junit</artifactId>
    <scope>test</scope>
</dependency>
```

Here is what each module brings to the table:

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle.
- **citrus-camel** provides the `camel:` URI scheme that lets Citrus send messages directly into Camel routes, plus CamelContext sharing between the frameworks.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.
- **camel-quarkus-junit** provides the `CamelQuarkusTestSupport` base class with helper methods like `getMockEndpoint()`.

No Kafka module, no HTTP module, no Testcontainers. Everything runs in-memory within a single JVM.

# Sharing the CamelContext

The foundation of this testing approach is sharing a single CamelContext across Quarkus, Camel, and Citrus:

```java
@QuarkusTest
@CitrusSupport
class QuarkusApplicationTest extends CamelQuarkusTestSupport
        implements TestActionSupport {

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

`@Inject` tells Quarkus to provide the CamelContext that manages all routes in the application. `@BindToRegistry` registers this same instance in the Citrus component registry. From that point on, when Citrus encounters a `camel:` endpoint URI, it resolves it against this shared context — which means it can reach any route, any direct endpoint, and any mock endpoint that the application defines.

The class extends `CamelQuarkusTestSupport`, which adds Camel-specific testing utilities. The most important one is `getMockEndpoint(String uri)`, which returns a reference to a mock endpoint from the shared CamelContext. This is how the test sets expectations and verifies message delivery.

# Writing the test

The test combines Citrus's send action with Camel's MockEndpoint assertion:

```java
@Test
void shouldHandleEvents() {
    MockEndpoint mockEndpoint = getMockEndpoint("mock:words-out");
    mockEndpoint.expectedBodiesReceived(">> HOWDY");

    runner.when(
        send()
            .fork(true)
            .endpoint("camel:direct:words-in")
            .message()
            .body("Howdy")
    );

    runner.then(
        context -> {
            try {
                mockEndpoint.assertIsSatisfied();
            } catch (InterruptedException e) {
                throw new CitrusRuntimeException(
                    "Failed to verify mock endpoint", e);
            }
        }
    );
}
```

Let's walk through each part.

## Setting up the expectation

```java
MockEndpoint mockEndpoint = getMockEndpoint("mock:words-out");
mockEndpoint.expectedBodiesReceived(">> HOWDY");
```

Before sending any message, the test retrieves the `mock:words-out` endpoint from the shared CamelContext and declares what it expects to receive. The `expectedBodiesReceived(">> HOWDY")` call tells the mock to expect exactly one message with that body. If a different body arrives, or no message arrives at all, the assertion will fail.

Camel's MockEndpoint API is flexible. Beyond `expectedBodiesReceived()` you can assert on message count (`expectedMessageCount()`), headers (`expectedHeaderReceived()`), message ordering, and custom predicates. For this simple route, a body check is all we need.

## Sending the message

```java
runner.when(
    send()
        .fork(true)
        .endpoint("camel:direct:words-in")
        .message()
        .body("Howdy")
);
```

The **when** block sends the string `"Howdy"` to the `camel:direct:words-in` endpoint. Citrus parses the `camel:` prefix, looks up the CamelContext from its registry, resolves `direct:words-in` as a Camel endpoint, and delivers the message.

The `fork(true)` option is important here. The `direct:` component is synchronous — the route executes in the caller's thread, and the send action blocks until the route completes. Without forking, the test thread would be tied up inside the route, and the verification step below would never execute. Forking sends the message in a separate thread so the test can proceed to the assertion.

This is specific to `direct:` endpoints. Asynchronous transports like Kafka or JMS do not need forking because the send action returns immediately after publishing the message.

## Verifying the mock endpoint

```java
runner.then(
    context -> {
        try {
            mockEndpoint.assertIsSatisfied();
        } catch (InterruptedException e) {
            throw new CitrusRuntimeException(
                "Failed to verify mock endpoint", e);
        }
    }
);
```

The **then** block calls Camel's `assertIsSatisfied()` on the mock endpoint. This method checks that all previously declared expectations were met — in this case, that exactly one message with body `">> HOWDY"` arrived. If the expectation is not satisfied within a timeout (default 10 seconds), the assertion fails with a detailed message showing what was expected versus what was received.

This hybrid approach — Citrus for test orchestration and message sending, Camel MockEndpoint for verification — gives you the best of both frameworks. Citrus provides the readable Given-When-Then structure and multi-protocol support, while Camel provides deep introspection into route behavior.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events:

1. **Quarkus starts** the application in test mode.
2. **Apache Camel discovers** the `RouteBuilder` and registers the `direct:words-in` → `mock:words-out` route.
3. **The CamelContext** is injected into the test and registered with Citrus.
4. **The test retrieves** the `mock:words-out` endpoint and sets the expected body.
5. **Citrus sends** `"Howdy"` to `camel:direct:words-in` in a forked thread.
6. **The Camel route** transforms the message to `">> HOWDY"` and sends it to `mock:words-out`.
7. **The mock endpoint** captures the message.
8. **`assertIsSatisfied()`** verifies the captured message matches the expectation.

The entire test runs in-memory with no external infrastructure. Typical execution time is under two seconds, making this approach ideal for fast feedback during development.

# When to use direct testing vs external endpoints

This direct-endpoint approach and the broker-based approach shown in the other samples on this site serve different purposes. Both are valuable, and many projects benefit from using both.

**Use direct testing when:**
- You want fast feedback on route transformation logic
- You are developing routes before external systems are available
- You need tests that run in CI without Docker
- You are testing complex routing patterns (content-based routing, splitting, aggregation) in isolation

**Use external endpoint testing when:**
- You need to verify end-to-end integration with a real broker
- You are testing protocol-specific behavior (JMS transactions, Kafka consumer groups)
- You want to validate the production configuration
- You need to test timing, ordering, or concurrency under realistic conditions

The direct approach tests what the route *does*. The external approach tests that the route *works* with real infrastructure. The [Camel JMS sample](/samples/camel-quarkus-jms/), [Camel Kafka sample](/samples/camel-quarkus-kafka/), and [Camel MQTT sample](/samples/camel-quarkus-mqtt/) on this site demonstrate the external approach.

# Where to go from here

This example covers the simplest possible route, but the same testing pattern scales to more complex scenarios:

- **Content-based routing**: Add a `choice().when()` block to the route and write separate tests for each branch, verifying that messages land on the correct mock endpoint based on content.
- **Multiple interconnected routes**: Use `direct:` endpoints to connect routes in a pipeline. Test each route independently with its own mock endpoint, then test the full pipeline end to end.
- **Error handling**: Add a `deadLetterChannel()` or `onException()` handler and verify that error messages reach a `mock:errors` endpoint when the route encounters invalid input.
- **Route advice**: Use Camel's `adviceWith()` API to replace production endpoints with mocks at test time, enabling direct testing of routes that normally use external transports.
- **Complex EIPs**: Test the Splitter, Aggregator, Recipient List, or any other enterprise integration pattern. Each pattern produces observable output that mock endpoints can capture and verify.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers Camel integration, mock endpoints, and the many other transports that Citrus supports.
