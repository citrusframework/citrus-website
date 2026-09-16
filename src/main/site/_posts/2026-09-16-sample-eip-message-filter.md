---
layout: sample
title: Testing the Message Filter Pattern with Citrus
name: message-filter
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Message Filter EIP in Apache Camel with Citrus across Quarkus, Spring Boot and YAML DSL
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/message-filter/
---

At some point applications may need to selectively accept messages for processing.
In other words not all messages on a destination should be processed based on a filter criteria.
A notification service subscribed to an order stream may only care about high-value purchases — orders above a certain threshold — while lower-value orders get batched into a daily digest.
A monitoring pipeline might only forward alerts that exceed a severity level.
A compliance service might only inspect transactions from specific regions.

In all these cases, the goal is the same: inspect each message against a predicate and let matching messages through while silently discarding the rest.
This is the [Message Filter](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Filter.html) pattern, one of the fundamental routing patterns described in *Enterprise Integration Patterns* by Hohpe and Woolf.

[Apache Camel](https://camel.apache.org) implements this pattern with the `filter()` EIP — a concise, declarative way to express pass-or-drop routing decisions.
But implementing the filter is only half the story.
How do you prove that matching messages actually reach the downstream channel?
And more importantly, how do you prove that non-matching messages are *not* forwarded?

Testing the absence of something is fundamentally harder than testing its presence.
You can wait for a message to arrive and assert its content, but waiting for a message that should never arrive requires a different strategy.
This is where [Citrus](https://citrusframework.org) shines: its `expectTimeout` action lets you assert that no message appears on a given endpoint within a defined time window — turning the absence of a message into a verifiable test outcome.

In this post, we build a complete example: a Camel Message Filter route that forwards only high-value orders, tested with Citrus on Quarkus, Spring Boot, and the Camel YAML DSL.

# The Camel route under test

The scenario is straightforward.
Orders arrive on a Kafka topic `eip.orders.placed`.
The route inspects each order's `amount` field and forwards only those with an amount of $100 or more to a `eip.orders.high-value` topic.
Orders below the threshold are silently dropped.

Here is the Camel route on Quarkus:

```java
@ApplicationScoped
public class MessageFilterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=filter-demo")
            .routeId("message-filter")
            .unmarshal().json()
            .filter(simple("${body[amount]} >= 100"))
                .log("High-value order ${body[order_id]}: $${body[amount]}")
                .marshal().json()
                .to("kafka:eip.orders.high-value?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
            .end();
    }
}
```

The route reads from the `eip.orders.placed` topic, unmarshals the JSON body into a map, and applies the filter predicate.
The `simple("${body[amount]} >= 100")` expression evaluates the `amount` field against the threshold.
Messages that pass the predicate are marshalled back to JSON and forwarded to the `eip.orders.high-value` topic.
Messages that fail the predicate simply fall through — Camel's `filter()` does nothing with them, and they are effectively discarded.

On Spring Boot, the route is identical except for the annotation:

```java
@Component
public class MessageFilterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=filter-demo")
            .routeId("message-filter")
            .unmarshal().json()
            .filter(simple("${body[amount]} >= 100"))
                .log("High-value order ${body[order_id]}: $${body[amount]}")
                .marshal().json()
                .to("kafka:eip.orders.high-value?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
            .end();
    }
}
```

The same routing logic expressed in the Camel YAML DSL looks like this:

```yaml
- route:
    id: message-filter
    from:
      uri: "kafka:eip.orders.placed"
      parameters:
        brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
        groupId: filter-demo
      steps:
        - unmarshal:
            json:
              library: Jackson
        - filter:
            simple: "${body[amount]} >= 100"
            steps:
              - log: "High-value order ${body[order_id]}: $${body[amount]}"
              - marshal:
                  json:
                    library: Jackson
              - to:
                  uri: "kafka:eip.orders.high-value"
                  parameters:
                    brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
```

All three variants express the same intent: pass high-value orders, drop the rest.
The difference between a Message Filter and a one-branch Content-Based Router is purely semantic — `filter()` communicates intent better than a `choice()` with a single `when` clause — but the testing challenge is the same: you need to verify both what the filter lets through and what it blocks.

# Setting up the test infrastructure

The route consumes its messages from a Kafka topic `eip.orders.placed`.
This means before any test can run, we need a Kafka broker.
Citrus can be used to provision the test infrastructure as part of the test setup.
Citrus manages this lifecycle with Testcontainers — a Docker Compose file defines the Kafka setup, and Citrus starts it before the test suite and tears it down after.

The Compose file provisions a single-node Kafka broker in KRaft mode (no ZooKeeper) alongside a Kafka UI for visual debugging:

```yaml
services:
  kafka:
    image: docker.io/apache/kafka:4.3.1
    ports:
      - "9092:9092"
    environment:
      - KAFKA_NODE_ID=1
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - KAFKA_AUTO_CREATE_TOPICS_ENABLE=true
    healthcheck:
      test: ["CMD-SHELL", "nc -z localhost 9092"]
      interval: 5s
      timeout: 5s
      retries: 12
      start_period: 30s
```

The Citrus infrastructure setup ties this Compose file to the test lifecycle.
On Quarkus, this uses the `@CitrusConfiguration` annotation with `@BindToRegistry`:

```java
@CitrusConfiguration
public class EipInfraSetup implements TestActionSupport {

    @BindToRegistry
    public BeforeSuite startInfra() {
        return beforeSuite().actions(
                    testcontainers().compose()
                            .up("_infra/compose.yaml")
                            .containerName("eip-infra")
                            .autoRemove(false),
                    waitFor()
                            .http()
                            .url("http://localhost:8090")
                            .seconds(25)
                ).build();
    }

    @BindToRegistry
    public AfterSuite stopInfra() {
        return afterSuite().actions(
                    camel().camelContext().stop(),
                    testcontainers().compose()
                            .down()
                            .containerName("eip-infra")
                ).build();
    }
}
```

On Spring Boot, the same logic uses Spring's `@Configuration` and `@Bean`:

```java
@Configuration
public class EipInfraSetup implements TestActionSupport {

    @Bean
    public BeforeSuite startInfra() {
        return beforeSuite().actions(
                    testcontainers().compose()
                            .up("_infra/compose.yaml")
                            .containerName("eip-infra")
                            .autoRemove(false),
                    waitFor()
                            .http()
                            .url("http://localhost:8090")
                            .seconds(25)
                ).build();
    }

    @Bean
    public AfterSuite stopInfra() {
        return afterSuite().actions(
                    camel().camelContext().stop(),
                    testcontainers().compose()
                            .down()
                            .containerName("eip-infra")
                ).build();
    }
}
```

The test actions are identical across both runtimes — `testcontainers().compose().up(...)` starts the containers and `testcontainers().compose().down()` tears them down.
Only the bean registration mechanism differs.

# The message template

All tests share a single JSON message template stored in `src/test/resources/templates/order.json`.
Citrus templates use `${variable}` placeholders that are resolved at runtime from test variables:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-${id}",
  "item_sku": "SKU-${id}",
  "quantity": 1,
  "amount": ${amount},
  "destination_country": "${country}",
  "contains_hazmat": ${hazmat},
  "shipping_priority": "STANDARD"
}
```

This template serves double duty: Citrus uses it to construct the message body when sending, and as the expected body when receiving.
By changing just the `${amount}` variable between tests, we control whether the order passes or fails the filter — same template, different outcome.

# Waiting for the Camel route

Camel routes that consume from Kafka take a moment to start and connect to the broker.
If a test sends a message before the route's consumer is ready, the message sits unprocessed and the test times out.

The example addresses this with a reusable utility interface that uses Camel's Control Bus to poll the route status:

```java
public interface EipTestSupport extends TestActionSupport {

    default TestActionBuilder<?> waitForCamelRouteStarted(
            String routeId, CamelContext camelContext) {
        return repeatOnError()
                .until((i, context) -> i > 20)
                .autoSleep(Duration.ofSeconds(1))
                .actions(
                    camel().camelContext(camelContext)
                            .controlBus()
                            .route(routeId)
                            .status()
                            .result(ServiceStatus.Started),
                    sleep().seconds(5)
                );
    }
}
```

This action repeatedly queries the Camel context for the route's status.
It retries up to 20 times with a one-second sleep between attempts, and once the route reports `Started`, it waits an additional 5 seconds for the Kafka consumer to fully connect.
Every test class implements this interface and calls `waitForCamelRouteStarted(...)` in its `given` phase before sending any messages.

# Testing the pass-through case

The first test verifies that a high-value order passes through the filter and arrives on the downstream topic.
We set `amount` to `250.00` — well above the $100 threshold — so the filter predicate evaluates to true.

```java
@Test
public void shouldPassHighValueOrder() {
    t.given(
        createVariables()
            .variable("id", "citrus:randomNumber(4)")
            .variable("amount", 250.00)
            .variable("country", "US")
            .variable("hazmat", false)
    );

    t.given(waitForCamelRouteStarted("message-filter", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .fork(true)
            .body(Resources.create("templates/order.json"))
            .header("kafka.KEY", "${id}")
    );

    t.then(
        receive()
            .endpoint("kafka:eip.orders.high-value?consumerGroup=citrus-high-value-group")
            .message()
            .body(Resources.create("templates/order.json"))
    );
}
```

The test follows Citrus's given-when-then structure.
The `given` phase creates test variables — including a random order ID generated by `citrus:randomNumber(4)` — and waits for the route to be ready.
The `when` phase sends a message to the input Kafka topic using the shared template.
The `then` phase receives the message from the output topic and validates its body against the same template.

The `fork=true` option on the send action makes sure to avoid running into racing conditions where the next `receive` action initializes the consumer on the Kafka topic too late.
There is a small but unpredictable delay between producing a message on the input topic and seeing it appear on the output topic — the Camel consumer needs to pick it up, process it, and the Kafka producer needs to publish it.
Depending on the performance of Citrus vs. Camel processing the output Kafka message might have been sent already before the Kafka consumer starts its offset.
With the fork option we make sure to start listening concurrently to sending the event that triggers the Camel route logic.

The receive action uses a dedicated consumer group (`citrus-high-value-group`) to avoid interfering with the application's own consumer groups.

# Testing the filter-out case

The second test is the more interesting one.
We send an order with `amount` set to `50.00` — below the $100 threshold — and need to verify that it does *not* appear on the downstream topic.

How do you test the absence of a message?
You wait for a reasonable period and assert that nothing arrived.
Citrus provides `expectTimeout` for exactly this purpose:

```java
@Test
public void shouldFilterLowValueOrder() {
    t.given(
        createVariables()
            .variable("id", "citrus:randomNumber(4)")
            .variable("amount", 50.00)
            .variable("country", "US")
            .variable("hazmat", false)
    );

    t.given(waitForCamelRouteStarted("message-filter", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .fork(true)
            .body(Resources.create("templates/order.json"))
            .header("kafka.KEY", "${id}")
    );

    t.then(
        expectTimeout()
            .endpoint("kafka:eip.orders.high-value?consumerGroup=citrus-filter-reject-group")
            .timeout(5000)
    );
}
```

The `expectTimeout` action listens on the `eip.orders.high-value` topic for 5 seconds.
If any message arrives during that window, the test *fails* — proving the filter accidentally let something through.
If the 5-second window elapses with no message, the test *passes* — confirming the filter correctly blocked the low-value order.

This is the inverse of a normal receive assertion: instead of "fail if no message arrives", it is "fail if a message *does* arrive".
The action uses a separate consumer group (`citrus-filter-reject-group`) so it does not compete with the pass-through test's consumer for messages.

The 5-second timeout is a pragmatic choice.
It needs to be long enough for a message to have arrived if it was going to — accounting for Kafka consumer lag and Camel processing time — but short enough that the test suite does not drag.
Five seconds is generous for a local Kafka broker processing a single-step filter route.

# Testing with the YAML DSL

The Camel CLI supports running routes from YAML files and testing them with Citrus YAML test definitions.
The Message Filter test in YAML DSL follows the same two-scenario structure:

```yaml
name: message-filter-test
description: >-
  Test verifying the Message Filter pattern — only high-value
  orders (amount >= 100) pass through
variables:
  - name: kafka.broker
    value: localhost:9092
actions:
  - testcontainers:
      compose:
        up:
          file: "_infra/compose.yaml"
  - camel:
      jbang:
        run:
          integration:
            name: "message-filter"
            file: "../message-filter.yaml"
            systemProperties:
              file: "../application.properties"

  # Scenario 1: Low-value order should be filtered out
  - createVariables:
      variables:
        - name: id
          value: "citrus:randomNumber(4)"
        - name: amount
          value: "50.00"
        - name: country
          value: "US"
        - name: hazmat
          value: "false"
  - send:
      endpoint: >-
        kafka:eip.orders.placed?server=${kafka.broker}
      message:
        body:
          resource:
            file: "templates/order.json"
  - expectTimeout:
      endpoint: >-
        kafka:eip.orders.high-value?server=${kafka.broker}&consumerGroup=citrus-filter-reject-group
      wait: 5000

  # Scenario 2: High-value order should pass the filter
  - createVariables:
      variables:
        - name: id
          value: "citrus:randomNumber(4)"
        - name: amount
          value: "250.00"
        - name: country
          value: "US"
        - name: hazmat
          value: "false"
  - send:
      endpoint: >-
        kafka:eip.orders.placed?server=${kafka.broker}
      message:
        body:
          resource:
            file: "templates/order.json"
  - receive:
      endpoint: >-
        kafka:eip.orders.high-value?server=${kafka.broker}&consumerGroup=citrus-high-value-group
      message:
        body:
          resource:
            file: "templates/order.json"
  - camel:
      jbang:
        verify:
          integration: "message-filter"
          logMessage: "High-value order"
```

The YAML test is self-contained: it starts the test infrastructure, launches the Camel integration with `camel:jbang:run`, runs both test scenarios, and verifies the Camel log output.
The final `camel:jbang:verify` action checks that the running integration logged the expected `"High-value order"` message, adding a secondary confirmation that the filter processed the matching order.

Notice how the YAML test reverses the scenario order compared to the Java tests: it tests the filter-out case first.
This is intentional — by verifying the `expectTimeout` before sending a high-value order, we ensure the output topic is clean and unambiguous for the subsequent pass-through assertion.

# Running on Quarkus and Spring Boot

The Camel route logic is identical across both runtimes.
The test code differences are in the wiring annotations, following a consistent pattern.

On Quarkus, the test class uses `@CitrusSupport` alongside `@QuarkusTest`, and the `CamelContext` is injected with CDI:

```java
@QuarkusTest
@CitrusSupport
class EipTests implements EipTestSupport {

    @CitrusResource
    TestCaseRunner t;

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    @Nested
    class MessageFilterTest {
        // ... test methods
    }
}
```

On Spring Boot, the test uses `@CitrusSpringSupport` alongside Spring Boot and Camel test annotations:

```java
@SpringBootTest(classes = RoutingFundamentalsApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class MessageFilterTest {

        @CitrusResource
        TestCaseRunner t;

        // ... test methods
    }
}
```

The Spring Boot variant requires explicit `@ContextConfiguration` to wire the Citrus configuration into the Spring application context.
The Quarkus variant discovers these automatically through CDI.
On Spring Boot, the `TestCaseRunner` is declared in the `@Nested` inner class, while on Quarkus it lives at the outer class level — a minor structural difference driven by how each test framework manages injection scopes.

Despite these wiring differences, the actual test methods — the `given`/`when`/`then` steps, the `send`, `receive`, and `expectTimeout` actions — are identical on both runtimes.
This is one of Citrus's design goals: the testing DSL stays the same regardless of the runtime environment.

# Test dependencies

Both runtime variants share the same core Citrus dependencies.
The runtime adapter is the only difference — `citrus-quarkus` for Quarkus, `citrus-spring` for Spring Boot:

```xml
<!-- Quarkus runtime adapter -->
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>

<!-- OR Spring Boot runtime adapter -->
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-spring</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Both runtimes share these common dependencies:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-camel</artifactId>
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
    <artifactId>citrus-validation-json</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

The `citrus-camel` module provides the Camel Control Bus integration used in `waitForCamelRouteStarted`.
The `citrus-kafka` module provides the Kafka send and receive actions.
The `citrus-testcontainers` module manages the Docker Compose lifecycle.
The `citrus-validation-json` module enables JSON message validation with the template-based assertions.

# Running the tests

Make sure Docker (or Podman) is running, then execute:

```bash
# Quarkus
cd examples/09-routing-fundamentals/quarkus
mvn verify

# Spring Boot
cd examples/09-routing-fundamentals/spring-boot
mvn verify
```

The test suite starts the Kafka broker via Testcontainers, boots the Camel application, runs both filter tests, and tears everything down.
The pass-through test completes in a few seconds as soon as the message arrives on the output topic.
The filter-out test takes at least 5 seconds — the full `expectTimeout` window — because Citrus must wait long enough to confirm no message appears.

For the YAML DSL variant, install the [Camel CLI](https://camel.apache.org/manual/camel-jbang.html) and run:

```bash
cd examples/09-routing-fundamentals/yaml-dsl
camel test test/message-filter.citrus.it.yaml
```

# Key takeaways

Testing the Message Filter pattern requires verifying two distinct outcomes: messages that should pass through, and messages that should be blocked.
The second case — proving the absence of a forwarded message — is the more challenging one.

Here is what Citrus brings to this problem:

- **`expectTimeout`** — Asserts that no message arrives on a given endpoint within a time window. This turns the absence of a message into a concrete, verifiable test assertion rather than relying on log inspection or manual checks.
- **`fork=true`** — Ensure to start the consumer offset early (concurrent to the send operation) in order to avoid racing conditions between Citrus Kafka consumer initialization and Camel route processing.
- **Shared message templates** — The same `order.json` template works for both producing and consuming. By changing only the `${amount}` variable, you control whether the order passes or fails the filter.
- **Random test data** — The `citrus:randomNumber(4)` function generates unique order IDs per test run, preventing collisions between repeated test executions.
- **Multi-runtime portability** — The same test DSL runs on Quarkus, Spring Boot, and the Camel CLI with YAML DSL. Only the wiring annotations change; the test logic stays identical.

The complete source code for this example is available on GitHub:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/spring-boot)
- [YAML DSL variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/yaml-dsl)

Give it a try, and let us know what you think!
