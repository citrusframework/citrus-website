---
layout: sample
title: Testing the Wire Tap Pattern with Citrus
name: wire-tap
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Wire Tap EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/wire-tap/
---

Production integration systems need observability.
Every order that flows through a pipeline should leave an audit trail — a timestamped record of what was processed, when, and by whom.
At the same time, auditing must never slow down or block the main processing flow.
If the audit system is overloaded, unreachable, or simply slow, the customer waiting for their order confirmation should not notice.

The naive approach is to add a synchronous `.to("kafka:eip.orders.audit")` step in the middle of the route.
But now the main flow waits for the Kafka acknowledgment on the audit topic before continuing.
A spike in audit topic latency — or a broker partition reassignment — stalls every order in the pipeline.

The [Wire Tap](https://www.enterpriseintegrationpatterns.com/patterns/messaging/WireTap.html) pattern, described in *Enterprise Integration Patterns* by Hohpe and Woolf, solves this by sending a *copy* of the message to a secondary channel without affecting the main flow.
The main route continues immediately; the tapped copy processes asynchronously in a separate thread.
Think of it as a network packet capture — it observes the traffic without interfering.

[Apache Camel](https://camel.apache.org) implements this with the `wireTap()` EIP, which sends a shallow copy of the current exchange to another endpoint in a separate thread.
The main route does not wait for the tap to complete, and a failure in the tap does not propagate back to the main flow.

Testing a wire tap creates a challenge that is distinct from other routing patterns.
With a content-based router or a splitter, you verify a single flow: message in, message out on the expected channel.
With a wire tap, a single inbound message produces activity on *two* independent paths — the main processing flow and the audit tap — and both must be verified.
The test needs to prove that the main flow completed normally *and* that the tapped copy reached the audit channel, even though the two paths run asynchronously.

This is where [Citrus](https://citrusframework.org) comes in.
Citrus can send a message to the input topic and then verify that both the main route and the audit route processed the exchange — proving that the wire tap fired without disrupting the primary flow.
In this post, we walk through a complete example that tests a Wire Tap route with Citrus, running on both Quarkus and Spring Boot.

# The Camel route under test

The scenario is an order processing pipeline with audit logging.
Orders arrive on a Kafka topic `eip.orders.processing` as JSON messages.
The main flow processes each order — stamping it with a `PROCESSED` status and a timestamp — then publishes the result to `eip.orders.processed`.
Along the way, a wire tap sends a copy of the original order to an audit route, which enriches it with audit metadata and publishes it to `eip.orders.audit`.

Here is the Camel route on Quarkus:

```java
@ApplicationScoped
public class WireTapRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.processing?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=wiretap-demo")
            .routeId("wire-tap-main")
            .unmarshal().json()
            .log("Processing order ${body[order_id]} — main flow")
            .wireTap("direct:audit")
            .process(exchange -> {
                var body = exchange.getIn().getBody(java.util.Map.class);
                body.put("status", "PROCESSED");
                body.put("processed_at", System.currentTimeMillis());
            })
            .log("Order ${body[order_id]} processed successfully")
            .marshal().json()
            .to("kafka:eip.orders.processed?brokers={% raw %}{{kafka.brokers}}{% endraw %}");

        from("direct:audit")
            .routeId("wire-tap-audit")
            .process(exchange -> {
                var body = new LinkedHashMap<>(
                        exchange.getIn().getBody(java.util.Map.class));
                exchange.getIn().setBody(body);
            })
            .log("AUDIT: Recording copy of order ${body[order_id]} for compliance")
            .process(exchange -> {
                var body = exchange.getIn().getBody(java.util.Map.class);
                body.put("audit_timestamp", System.currentTimeMillis());
                body.put("audit_source", "wire-tap");
            })
            .marshal().json()
            .to("kafka:eip.orders.audit?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

There are several things worth noting about this route.

First, the `.wireTap("direct:audit")` call sits between the unmarshal step and the main processing logic.
When Camel reaches this point, it sends a shallow copy of the current exchange to `direct:audit` in a separate thread and *immediately* continues to the next step in the main route.
The main flow does not wait for the audit route to complete.

Second, the audit route creates a deep copy of the message body with `new LinkedHashMap<>(...)` before modifying it.
This is critical.
The wire tap sends a *shallow* copy of the exchange — the body reference in the tapped copy points to the same `Map` object as the main flow.
Without the deep copy, modifications in the audit route (adding `audit_timestamp` and `audit_source`) would be visible in the main flow's body, corrupting it.
The deep copy isolates the two flows so they can modify the body independently.

Third, the two flows produce output on *different* Kafka topics.
The main flow writes to `eip.orders.processed` with a `PROCESSED` status.
The audit route writes to `eip.orders.audit` with additional audit metadata.
This separation means a test can verify both outputs independently.

On Spring Boot, the route is identical except for the annotation:

```java
@Component
public class WireTapRoute extends RouteBuilder {

    @Override
    public void configure() {
        // ... identical route logic
    }
}
```

# How the wire tap works

The wire tap is often confused with the multicast pattern, but they differ in fundamental ways:

| Dimension          | Wire Tap                                 | Multicast                                      |
|--------------------|------------------------------------------|------------------------------------------------|
| **Blocking**       | No — async, fire-and-forget              | Yes — waits for all recipients                 |
| **Failure impact** | Tap failure does not affect main flow    | Recipient failure can affect the flow          |
| **Threading**      | Tap runs in a separate thread            | Can run in parallel or sequential threads      |
| **Use case**       | Auditing, logging, analytics, monitoring | Sending to multiple recipients that all matter |

The wire tap is specifically designed for secondary concerns — things that should happen alongside the main flow but must never block or break it.
If the audit topic is temporarily unavailable, the main order processing continues without interruption.
The tapped message is lost, but the customer's order is not affected.

This fire-and-forget behavior is what makes wire taps valuable in production — and what makes them tricky to test.
In a test environment, you need to prove that the tap *did* fire, even though the main flow does not depend on it.

# What makes testing a wire tap different

Testing a wire tap is fundamentally different from testing other routing patterns because a single inbound message produces *two independent processing paths*.

With a content-based router, one message goes in and one message comes out on a deterministic channel.
With a splitter, one message goes in and N messages come out on the same channel.
With a wire tap, one message goes in and activity happens on *two separate channels concurrently*: the main processed output and the audit copy.

This creates three verification challenges:

- **Main flow completion**: the order must arrive on `eip.orders.processed` with a `PROCESSED` status, proving the main route ran to completion.
- **Audit flow completion**: the same order must also arrive on `eip.orders.audit` with audit metadata, proving the wire tap fired and the audit route processed the copy.
- **Independence**: the main flow must not be affected by the audit route's behavior. The test should verify both flows completed, but a slow or failing audit should not prevent the main flow from succeeding.

The asynchronous nature of the wire tap adds a timing dimension.
The audit route may complete before, after, or concurrently with the main flow.
The test must accommodate this non-deterministic ordering — it cannot assume the audit message appears before or after the processed message.

Citrus handles this with two techniques: a forked send that does not block the test thread, and a `parallel()` block that consumes from both output topics simultaneously — so neither receive blocks the other regardless of which flow completes first.

# The message templates

The test uses three JSON templates.

The input order template at `src/test/resources/templates/order.json`:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-00${id}",
  "item_sku": "SKU-TEST-${id}",
  "quantity": 1,
  "amount": ${amount},
  "destination_country": "${country}",
  "shipping_priority": "${priority}"
}
```

The processed order template at `src/test/resources/templates/processed-order.json`, used to validate the main flow's output:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-00${id}",
  "item_sku": "SKU-TEST-${id}",
  "quantity": 1,
  "amount": ${amount},
  "destination_country": "${country}",
  "shipping_priority": "${priority}",
  "status": "PROCESSED",
  "processed_at": "@ignore@"
}
```

The audit order template at `src/test/resources/templates/audit-order.json`, used to validate the audit flow's output:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-00${id}",
  "item_sku": "SKU-TEST-${id}",
  "quantity": 1,
  "amount": ${amount},
  "destination_country": "${country}",
  "shipping_priority": "${priority}",
  "status": "PROCESSED",
  "processed_at": "@ignore@",
  "audit_timestamp": "@ignore@",
  "audit_source": "wire-tap"
}
```

Notice the progression across the three templates.
The input order has the base fields.
The processed order adds `status` and `processed_at` — fields set by the main flow.
The audit order adds `audit_timestamp` and `audit_source` on top of the processed fields — metadata added by the audit route.

The `@ignore@` markers tell Citrus to accept any value for timestamp fields.
These are system-generated values (millisecond timestamps from `System.currentTimeMillis()`) that differ on every test run.
The `audit_source` field, however, is validated exactly: it must be the string `"wire-tap"`, confirming that the audit route — not some other process — produced this record.

# The wire tap test

The test sends a single order and verifies that both the main flow's output and the audit copy arrive on their respective Kafka topics with the expected content.
Here is the test on Quarkus:

```java
@Nested
class WireTapTest {

    @Test
    public void shouldProcessOrderAndSendAuditCopy() {
        t.given(
            createVariables()
                .variable("id", "citrus:randomNumber(4)")
                .variable("amount", "150.00")
                .variable("country", "US")
                .variable("priority", "EXPRESS")
        );

        t.given(waitForCamelRouteStarted("wire-tap-main", camelContext));
        t.given(waitForCamelRouteStarted("wire-tap-audit", camelContext));

        t.when(
            send()
                .endpoint("kafka:eip.orders.processing")
                .message()
                .fork(true)
                .body(Resources.create("templates/order.json"))
                .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
        );

        t.then(
            parallel()
                .actions(
                    receive()
                        .endpoint("kafka:eip.orders.processed?consumerGroup=citrus-wiretap-processed-group")
                        .message()
                        .body(Resources.create("templates/processed-order.json")),
                    receive()
                        .endpoint("kafka:eip.orders.audit?consumerGroup=citrus-wiretap-audit-group")
                        .message()
                        .body(Resources.create("templates/audit-order.json"))
                )
        );
    }
}
```

Let us walk through each phase.

## The given phase — waiting for two routes

The `given` phase creates test variables and then waits for *two* routes to be fully started: `wire-tap-main` and `wire-tap-audit`.

This is different from single-route patterns where you wait for one route.
With a wire tap, the main route's Kafka consumer may be ready before the audit route's `direct:` endpoint is registered.
If the test sends a message before both routes are started, the wire tap call might fail silently — the message enters the main flow but the tapped copy has nowhere to go.
Waiting for both routes ensures the full pipeline is ready.

## The when phase — a forked send

The `when` phase sends a single order to `eip.orders.processing`.
The `KafkaMessageHeaders.MESSAGE_KEY` is set to the random `${id}` for consistent partitioning.

The `.fork(true)` on the send action is important.
By default, a Citrus `send()` action blocks the test thread until the Kafka producer confirms delivery.
With `fork(true)`, the send executes in a background thread and the test immediately proceeds to the next phase.
This matters because the `then` phase needs to start consuming from both output topics *before* the Camel route finishes processing — otherwise a fast route might publish its output before the Citrus consumers are subscribed, and the `receive()` action would miss the message.

Unlike the load balancer test (which sends multiple messages to prove distribution), the wire tap test needs only *one* message.
A single message should trigger both flows — the main processing and the audit copy.
If the wire tap is misconfigured (for example, commented out or pointing to a nonexistent endpoint), one message is enough to detect the problem.

## The then phase — parallel receives on two topics

The `then` phase is where the wire-tap-specific verification happens.
A `parallel()` block wraps two `receive()` actions that run concurrently:

1. One `receive()` consumes from `eip.orders.processed` and validates the message against the `processed-order.json` template — proving the main flow ran to completion, stamped the order with `PROCESSED` status and a timestamp.
2. The other `receive()` consumes from `eip.orders.audit` and validates against the `audit-order.json` template — proving the wire tap fired and the audit route enriched the copy with `audit_timestamp` and `audit_source`.

The `parallel()` wrapper is essential here.
If the receives ran sequentially, one would block waiting for its message while the other topic's message sits unconsumed.
Because the wire tap is asynchronous, the audit message might arrive before or after the processed message — sequential receives would depend on a specific ordering that is not guaranteed.
Running both receives in parallel eliminates this timing dependency: whichever message arrives first is consumed immediately, and neither receive blocks the other.

Each receive uses a dedicated consumer group (`citrus-wiretap-processed-group` and `citrus-wiretap-audit-group`) to avoid interference with other tests or the application's own consumers.

If the wire tap is removed from the route, the receive on `eip.orders.audit` times out and the test fails — no audit message was produced.
If the audit route corrupts the message (for example, due to the shallow copy body mutation problem), the template validation catches the content mismatch.
If the main flow fails, the receive on `eip.orders.processed` times out.

# Why parallel receives beat route counter assertions

An earlier version of this test used Camel's `ManagedRouteMBean` API to check exchange counters — verifying that both the `wire-tap-main` and `wire-tap-audit` routes had processed at least one exchange.
That approach confirms the routes *ran*, but it says nothing about *what they produced*.

The parallel-receive approach is stronger in three ways:

- **Content validation**: the test proves that the processed order has a `PROCESSED` status and the audit order has `audit_source: "wire-tap"`. Route counters cannot verify message content.
- **End-to-end coverage**: consuming from the output Kafka topics proves the entire pipeline works — from Kafka input through Camel routing through JSON serialization to Kafka output. Counter-based assertions only prove the route's internal processing ran.
- **No dependency on Camel internals**: the test uses Kafka topics as its interface, not Camel's JMX management beans. This makes it portable — the same assertion approach works regardless of whether the route is deployed on Camel, a different integration framework, or a plain Kafka Streams application.

# Waiting for the Camel routes

Camel routes that consume from Kafka take a moment to start and subscribe to the broker.
If the test sends a message before the route's consumer is ready, the message sits unprocessed and the assertions time out.

The example addresses this with a reusable utility that uses Camel's Control Bus to poll the route status:

```java
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
```

This polls the Camel context up to 20 times with one-second pauses.
Once the route reports `Started`, it waits an additional five seconds for the Kafka consumer to fully connect and receive its partition assignments.
The wire tap test calls this utility *twice* — once for `wire-tap-main` and once for `wire-tap-audit` — ensuring both routes are ready before sending any messages.

# Setting up the test infrastructure

The route consumes from Kafka, so the test needs a running broker.
Citrus manages this lifecycle with Testcontainers — a Docker Compose file defines a single-node Kafka (KRaft mode) setup along with a Kafka UI for observability.

On Quarkus, the infrastructure setup uses `@CitrusConfiguration` with `@BindToRegistry`:

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

On Spring Boot, the same logic uses `@Configuration` and `@Bean`:

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

The test actions are identical — only the bean registration mechanism differs between the two runtimes.

# Running on Quarkus and Spring Boot

The Camel route logic is identical across both runtimes.
The test code differences are limited to the wiring annotations.

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
    class WireTapTest {
        // ... test methods
    }
}
```

On Spring Boot, the test uses `@CitrusSpringSupport` alongside Spring Boot and Camel test annotations:

```java
@SpringBootTest(classes = AdvancedRoutingApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class WireTapTest {

        @CitrusResource
        TestCaseRunner t;

        // ... test methods
    }
}
```

On Spring Boot, the `TestCaseRunner` is declared in the `@Nested` inner class, while on Quarkus it lives at the outer class level — a structural difference driven by how each test framework manages injection scopes.
The actual test methods inside `WireTapTest` are identical on both runtimes.

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

The `citrus-camel` module provides the Control Bus integration used in `waitForCamelRouteStarted`.
The `citrus-kafka` module provides the Kafka send and receive actions used to produce and consume messages on the input, processed, and audit topics.
The `citrus-testcontainers` module manages the Docker Compose lifecycle for the Kafka broker.
The `citrus-validation-json` module enables JSON message validation with template-based assertions and `@ignore@` matchers used to verify the processed and audit output messages.

# Running the tests

Make sure Docker (or Podman) is running, then execute:

```bash
# Quarkus
cd examples/11-advanced-routing/quarkus
mvn verify

# Spring Boot
cd examples/11-advanced-routing/spring-boot
mvn verify
```

The test suite starts the Kafka broker via Testcontainers, boots the Camel application, runs the wire tap test (along with the other pattern tests in the suite), and tears everything down.

# Key takeaways

Testing a Wire Tap means verifying that a single inbound message triggers two independent processing paths — the main flow and the tapped copy — and that both complete successfully without interfering with each other.

Here is what Citrus brings to this problem:

- **Parallel receives for concurrent verification** — A `parallel()` block consumes from both `eip.orders.processed` and `eip.orders.audit` simultaneously. This eliminates timing dependencies between the asynchronous main flow and the wire tap — whichever message arrives first is consumed immediately, and neither receive blocks the other.
- **Forked send for non-blocking test flow** — The `.fork(true)` on the send action ensures the test thread proceeds to the receives before the Camel route finishes processing. Without forking, a fast route could publish output before the Citrus consumers are subscribed.
- **Content validation on both paths** — Each `receive()` validates the message body against a template. The processed order must have `status: "PROCESSED"`, and the audit order must have `audit_source: "wire-tap"`. This proves not just that both routes ran, but that they produced the correct output.
- **Multi-route startup coordination** — The `given` phase waits for *both* `wire-tap-main` and `wire-tap-audit` to report `Started`, ensuring the full pipeline is ready before sending messages. This prevents false negatives from messages arriving before the audit route is registered.
- **Multi-runtime portability** — The same test logic runs on Quarkus and Spring Boot. Only the wiring annotations change; the send, receive, and validation logic stays identical.

The complete source code for this example is available on GitHub:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/11-advanced-routing/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/11-advanced-routing/spring-boot)

Give it a try, and let us know what you think!
