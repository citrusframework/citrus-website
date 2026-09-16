---
layout: sample
title: Testing Message Reliability with Citrus
name: message-reliability
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing Dead Letter Queue patterns in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/message-reliability/
---

Reliability patterns are the safety nets of any messaging system.
When a message cannot be processed — because the payload is malformed, a downstream service is down, or the processing logic throws an unexpected exception — the system must handle the failure gracefully.
[Apache Camel](https://camel.apache.org) provides the Dead Letter Queue pattern for exactly this purpose: failed messages retry a configurable number of times, and when all retries are exhausted the original message is routed to a dedicated dead letter queue (DLQ) instead of being silently lost.

But here is the challenge: how do you *prove* that this safety net actually works?
A unit test that mocks the error handler tells you nothing about what happens when a real Kafka message fails processing three times in a row.
You need an integration test that sends a message into a live Camel route, lets the route retry and fail, and then verifies that the original message lands on the DLQ topic.

This is where [Citrus](https://citrusframework.org) comes in.
Citrus is an integration testing framework that speaks messaging natively — it can produce and consume Kafka messages, manage test infrastructure with Testcontainers, and wait for asynchronous outcomes like delayed DLQ deliveries.
In this post, we walk through a complete example that tests a Dead Letter Queue route with Citrus, running on both Quarkus and Spring Boot.

# The Camel route under test

The example application processes orders from a Kafka topic.
Most orders succeed and are forwarded to a processed topic.
Every order whose `order_id` is divisible by 5 throws a simulated exception, triggering the Dead Letter Queue.

Here is the Camel route:

```java
@ApplicationScoped
public class DeadLetterChannelRoute extends RouteBuilder {

    @Override
    public void configure() {
        errorHandler(deadLetterChannel("kafka:eip.orders.dlq?brokers={{kafka.brokers}}")
            .maximumRedeliveries(3)
            .redeliveryDelay(1000)
            .retryAttemptedLogLevel(LoggingLevel.WARN)
            .logExhausted(true)
            .useOriginalMessage());

        from("kafka:eip.orders.placed?brokers={{kafka.brokers}}&groupId=reliability-demo")
            .routeId("dead-letter-channel-demo")
            .unmarshal().json()
            .log("Processing order ${body[order_id]}")
            .process(exchange -> {
                var body = exchange.getIn().getBody(java.util.Map.class);
                long orderId = ((Number) body.get("order_id")).longValue();
                if (orderId % 5 == 0) {
                    throw new RuntimeException(
                        "Simulated failure for order " + orderId);
                }
            })
            .log("Order ${body[order_id]} processed successfully")
            .marshal().json()
            .to("kafka:eip.orders.processed?brokers={{kafka.brokers}}");

        from("kafka:eip.orders.dlq?brokers={{kafka.brokers}}&groupId=dlq-monitor")
            .routeId("dlq-monitor")
            .log("DLQ received failed order: ${body}")
            .log("Failure reason: ${header.CamelExceptionCaught}");
    }
}
```

There are three important things to notice in this route.

First, the `errorHandler` is configured with `maximumRedeliveries(3)` and `redeliveryDelay(1000)`.
This means a failing message will be retried three times, each with a one-second delay, before Camel gives up and routes it to `eip.orders.dlq`.
For a test, this translates to a minimum wait of about three seconds before the DLQ message appears — the test must account for this delay.

Second, `useOriginalMessage()` tells Camel to send the message to the DLQ exactly as it was *before* any processing.
Without this option, the DLQ would receive whatever partially-transformed state the message was in when the exception was thrown.
For our tests, this means the DLQ message will have the same JSON structure as the original input — we can reuse the same message template for both send and receive assertions.

Third, the route includes a `dlq-monitor` that consumes from the DLQ topic and logs each failure.
This is a monitoring concern, separate from the error handling itself, but it demonstrates a common pattern in production systems.

# Setting up the test infrastructure

Before any test can run, we need a Kafka broker.
Citrus manages this infrastructure lifecycle with Testcontainers — a Docker Compose file defines the Kafka setup, and Citrus starts it before the test suite and tears it down after.

The Compose file provisions a single-node Kafka broker in KRaft mode (no ZooKeeper):

```yaml
services:
  kafka:
    image: docker.io/apache/kafka:latest
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

The Citrus infrastructure setup class ties this Compose file to the test lifecycle.
It starts the containers before all tests and stops both the Camel context and the containers after all tests.

On Quarkus, this uses the `@CitrusConfiguration` annotation and Citrus registry bindings:

```java
@CitrusConfiguration
public class EipInfraSetup implements TestActionSupport {

    @BindToRegistry
    public BeforeSuite startInfra() {
        return beforeSuite().actions(
                    testcontainers().compose()
                            .up("_infra/compose.yaml")
                            .containerName("eip-infra")
                            .autoRemove(false)
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

On Spring Boot, the same logic uses Spring's `@Configuration` and `@Bean` annotations:

```java
@Configuration
public class EipInfraSetup implements TestActionSupport {

    @Bean
    public BeforeSuite startInfra() {
        return beforeSuite().actions(
                    testcontainers().compose()
                            .up("_infra/compose.yaml")
                            .containerName("eip-infra")
                            .autoRemove(false)
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

The test actions themselves are identical — `testcontainers().compose().up(...)` and `testcontainers().compose().down()` — only the bean registration mechanism differs between runtimes.

# The message template

All tests in this example share a single JSON message template.
Citrus templates use `${variable}` placeholders that are resolved at runtime from test variables, keeping the payload definition clean and reusable:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-00${id}",
  "item_sku": "SKU-SHIP-${id}",
  "quantity": 1,
  "amount": ${amount},
  "status": "${status}",
  "shipping_priority": "${priority}"
}
```

This template is stored in `src/test/resources/templates/order.json` and referenced in tests with `Resources.create("templates/order.json")`.
The same template works for both producing and consuming messages: when Citrus sends a message, the placeholders are filled with the test's variable values; when Citrus receives a message, the expected body uses the same template with the same variables, creating a clean round-trip assertion.

# Waiting for the Camel route

Camel routes that consume from Kafka take a moment to start and connect to the broker.
If a test sends a message before the route's consumer is ready, the message sits unprocessed on the topic and the test times out.

The example addresses this with a reusable `waitForCamelRouteStarted` utility that uses Camel's Control Bus to poll the route status:

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

# Testing the happy path

The first test verifies that a valid order is processed successfully and forwarded to the output topic.
We use `order_id=1001` — not divisible by 5, so the route will process it without errors.

```java
@Test
public void shouldProcessOrderSuccessfully() {
    t.given(
        createVariables()
            .variable("id", 1001)
            .variable("amount", 100)
            .variable("status", "placed")
            .variable("priority", "STANDARD")
    );

    t.given(waitForCamelRouteStarted("dead-letter-channel-demo", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .fork(true)
            .body(Resources.create("templates/order.json"))
            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
    );

    t.then(
        receive()
            .endpoint("kafka:eip.orders.processed?consumerGroup=citrus-processed-group")
            .message()
            .body(Resources.create("templates/order.json"))
            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
    );
}
```

The test follows Citrus's given-when-then structure.
The `given` phase sets up test variables and waits for the route to be ready.
The `when` phase sends an order message to the input Kafka topic.
The `then` phase receives and validates the processed message on the output topic.

There is one critical detail: `fork(true)` on the send action.
This tells Citrus to send the message and immediately proceed to the receive action without waiting for the Kafka acknowledgement to complete first.
Without `fork(true)`, there is a timing risk: the test sends the message, waits for the Kafka producer acknowledgement, and *then* starts listening on the output topic — but by that time the Camel route may have already processed the message and published it.
If the Citrus consumer starts listening after the output message was already published, the test misses it and times out.
The `fork(true)` option eliminates this race condition by overlapping the send and receive operations.

The receive action uses a dedicated consumer group (`citrus-processed-group`) to avoid interfering with the application's own consumer groups.
It verifies both the message body — using the same template with the same variable values — and the Kafka message key.

# Testing the failure path

The second test is where the Dead Letter Queue proves its value.
We send an order with `order_id=1000` — divisible by 5, which triggers the simulated exception in the route's processor.

```java
@Test
public void shouldRouteFailedOrderToDLQ() {
    t.given(
        createVariables()
            .variable("id", 1000)
            .variable("amount", 200)
            .variable("status", "placed")
            .variable("priority", "EXPRESS")
    );

    t.given(waitForCamelRouteStarted("dead-letter-channel-demo", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .fork(true)
            .message()
            .body(Resources.create("templates/order.json"))
            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
    );

    t.then(
        repeatOnError()
            .until((i, context) -> i > 10)
            .autoSleep(Duration.ofSeconds(1))
            .actions(
                receive()
                    .endpoint("kafka:eip.orders.dlq?consumerGroup=citrus-dlq-group")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            )
    );
}
```

The key difference from the happy path test is the `repeatOnError` wrapper around the DLQ receive action.
The Dead Letter Queue does not route the message to the DLQ immediately — it first retries 3 times with 1-second delays.
That means at least 3 seconds pass between the initial send and the moment the message appears on the DLQ topic.
The `repeatOnError` action handles this by polling: it attempts the receive up to 10 times, sleeping 1 second between attempts, until it finds the expected message.

This polling approach is intentional.
A fixed `sleep(5000)` would also work in most cases, but it is fragile — if the retry configuration changes, the test breaks.
The `repeatOnError` pattern adapts to the actual timing: it checks early and often, succeeding as soon as the message arrives regardless of the exact delay.

Notice that the DLQ receive uses the same message template as the original send.
Because the route configures `useOriginalMessage()`, the DLQ message is the untransformed input — same `order_id`, same `amount`, same `status`.
This is both a correctness property of the route and a testing convenience: we can assert the full message body without worrying about intermediate transformations.

# Running on Quarkus and Spring Boot

The Camel route logic is identical across both runtimes — the `DeadLetterChannelRoute` class differs only in its class annotation.
On Quarkus, the route uses `@ApplicationScoped`:

```java
@ApplicationScoped
public class DeadLetterChannelRoute extends RouteBuilder { ... }
```

On Spring Boot, it uses `@Component`:

```java
@Component
public class DeadLetterChannelRoute extends RouteBuilder { ... }
```

The test code differences are slightly more involved but follow a consistent pattern.
On Quarkus, the test class uses Citrus's `@CitrusSupport` annotation alongside `@QuarkusTest`, and the `CamelContext` is injected with CDI:

```java
@QuarkusTest
@CitrusSupport
class EipTests implements EipTestSupport {

    @CitrusResource
    TestCaseRunner t;

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    // ... test methods
}
```

On Spring Boot, the test uses `@CitrusSpringSupport` alongside Spring Boot and Camel test annotations, and the `CamelContext` is injected with `@Autowired`:

```java
@SpringBootTest(classes = ReliabilityApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class DeadLetterChannelRouteTest {

        @CitrusResource
        TestCaseRunner t;

        // ... test methods
    }
}
```

The Spring Boot variant requires explicit `@ContextConfiguration` to wire the Citrus configuration and the infrastructure setup class into the Spring application context.
The Quarkus variant discovers these automatically through CDI.

Despite these wiring differences, the actual test methods — the `given`/`when`/`then` steps, the Kafka send and receive actions, the `repeatOnError` polling — are identical on both runtimes.
This is one of Citrus's design goals: the testing DSL stays the same regardless of the runtime environment, so you can port tests between runtimes without rewriting the test logic.

# Test dependencies

Both runtime variants use the same set of Citrus test dependencies.
For Quarkus, you need `citrus-quarkus` as the runtime adapter:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

For Spring Boot, you need `citrus-spring` instead:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-spring</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Both runtimes share these common Citrus dependencies:

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

To run the tests, make sure Docker (or Podman) is running, then execute:

```bash
# Quarkus
cd examples/05-reliability/quarkus
mvn verify

# Spring Boot
cd examples/05-reliability/spring-boot
mvn verify
```

The test suite will start the Kafka broker via Testcontainers, boot the Camel application, run both test cases, and tear everything down.
The happy path test completes in a few seconds.
The DLQ test takes longer — at least 3-4 seconds for the retries to exhaust plus the polling overhead — but the `repeatOnError` wrapper keeps the wait time to a minimum.

# Key takeaways

Testing reliability patterns requires a different mindset than testing happy-path routing.
You need to deliberately trigger failures, wait for asynchronous error handling to complete, and verify that messages end up in the right place after retries are exhausted.

Here is what Citrus brings to this problem:

- **Deterministic test data** — By controlling the `order_id` value (1001 for success, 1000 for failure), you choose exactly which code path the route takes. No randomness, no flaky tests.
- **`fork(true)`** — Overlaps the Kafka send and receive operations to prevent timing races in asynchronous messaging tests.
- **`repeatOnError`** — Polls for delayed outcomes like DLQ messages without resorting to brittle fixed-duration sleeps.
- **Shared message templates** — The same `order.json` template works for both producing and consuming, with Citrus variables providing different values per test.
- **Multi-runtime portability** — The same test DSL runs on both Quarkus and Spring Boot. Only the wiring annotations change; the test logic stays identical.

The complete source code for this example is available on GitHub in both runtime variants:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/05-reliability/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/05-reliability/spring-boot)

Give it a try, and let us know what you think!
