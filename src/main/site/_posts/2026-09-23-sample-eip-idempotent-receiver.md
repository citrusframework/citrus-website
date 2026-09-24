---
layout: sample
title: Testing the Idempotent Receiver Pattern with Citrus
name: idempotent-receiver
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Idempotent Receiver EIP in Apache Camel with Citrus across Quarkus, Spring Boot and YAML DSL
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/idempotent-receiver/
---

Every decoupled messaging system that guarantees an at-least-once delivery has the same uncomfortable truth: your consumer *will* see the same message more than once.
A Kafka rebalance during processing, a consumer crash after business logic completes but before the offset is committed, a network hiccup triggering a producer retry — all of these produce duplicates.
For a notification service, a duplicate means a customer receives the same "your order shipped" email twice.
Annoying, but harmless.
For a payment service, a duplicate means the customer is charged twice.
That is a production incident.

The [Idempotent Receiver](https://www.enterpriseintegrationpatterns.com/patterns/messaging/IdempotentReceiver.html) pattern, described by Hohpe and Woolf, addresses this.
An idempotent receiver tracks which messages have already been processed and silently skips duplicates.
It maintains a set of processed message IDs — typically backed by a database, Redis, or an in-memory store — and checks each incoming message against the set before processing it.

[Apache Camel](https://camel.apache.org) implements this with the `idempotentConsumer()` EIP.
You provide an expression that extracts the unique message key and a repository that persists the seen keys.
Camel handles the check-and-store logic internally: if the key is already in the repository, the message is skipped; otherwise, it is processed and the key is recorded.

Testing an idempotent receiver requires proving two complementary behaviors: that unique messages pass through normally, and that duplicate messages are silently dropped.
The second assertion is the harder one — you need to prove a *negative*, that something did *not* happen.
Integration tests with [Citrus](https://citrusframework.org) handle this naturally using timeout-based assertions that confirm no output appears for the duplicate.

## The scenario

An order processing pipeline receives order events on a Kafka topic `eip.orders.placed`.
Each order carries an `order_id` that uniquely identifies it.
The idempotent receiver route consumes from this topic, deduplicates orders by `order_id` using a JDBC-backed repository in PostgreSQL, and forwards unique orders to a downstream topic `eip.orders.deduplicated`.

If the same `order_id` arrives twice — because a producer retried, because a Kafka rebalance caused a redelivery, or because an upstream system replayed events — the second occurrence is silently dropped.
The downstream topic never sees it.

This is the scenario we need to test end-to-end: send two identical messages, verify that exactly one arrives on the output topic, confirm that the duplicate produced no output, and check that the route processed both exchanges without errors.

## The Camel route

The route is concise.
The interesting part is the combination of `idempotentConsumer()` with a `JdbcMessageIdRepository` — a production-grade deduplication store backed by PostgreSQL.

### Quarkus

```java
@ApplicationScoped
public class IdempotentReceiverRoute extends RouteBuilder {

    @Inject
    DataSource dataSource;

    @Override
    public void configure() {
        JdbcMessageIdRepository idempotentRepo =
            new JdbcMessageIdRepository(dataSource, "payment-dedup");

        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=idempotent-demo")
            .routeId("idempotent-receiver")
            .unmarshal().json()
            .idempotentConsumer(jsonpath("$.order_id"), idempotentRepo)
            .log("Processing unique order ${body[order_id]}")
            .marshal().json()
            .to("kafka:eip.orders.deduplicated?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

The route reads from `eip.orders.placed`, deserializes the JSON body, and passes it through `idempotentConsumer()`.
The expression `jsonpath("$.order_id")` extracts the deduplication key from the message body.
The `JdbcMessageIdRepository` stores seen keys in a PostgreSQL table named `camel_messageprocessed`, keyed by the processor name `"payment-dedup"` and the extracted `order_id`.

When a message with a previously seen `order_id` arrives, `idempotentConsumer()` silently drops it — the route processing after `idempotentConsumer()` never executes for the duplicate.
Only unique messages reach the `marshal().json()` and `to("kafka:eip.orders.deduplicated")` steps.

### Spring Boot

```java
@Component
public class IdempotentReceiverRoute extends RouteBuilder {

    @Autowired
    DataSource dataSource;

    @Override
    public void configure() {
        JdbcMessageIdRepository idempotentRepo =
            new JdbcMessageIdRepository(dataSource, "payment-dedup");

        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=idempotent-demo")
            .routeId("idempotent-receiver")
            .unmarshal().json()
            .idempotentConsumer(jsonpath("$.order_id"), idempotentRepo)
            .log("Processing unique order ${body[order_id]}")
            .marshal().json()
            .to("kafka:eip.orders.deduplicated?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

`@Component` replaces `@ApplicationScoped`, and `@Autowired` replaces `@Inject` — the route logic is identical.

### Choosing the idempotent repository

Camel provides several idempotent repository implementations.
The choice depends on your durability and performance requirements:

| Repository                       | Durability              | Speed    | Use case                                       |
|----------------------------------|-------------------------|----------|------------------------------------------------|
| `MemoryIdempotentRepository`     | None (lost on restart)  | Fastest  | Development, testing                           |
| `JdbcMessageIdRepository`        | Persistent              | Moderate | Production (when a relational DB is available) |
| `RedisIdempotentRepository`      | Persistent, distributed | Fast     | Production (recommended for high throughput)   |
| `InfinispanIdempotentRepository` | Persistent, clustered   | Fast     | High-availability clusters                     |

Our example uses `JdbcMessageIdRepository` because the application already has PostgreSQL in the stack for other purposes — no additional infrastructure required.
The repository needs a table to store processed message IDs:

```sql
CREATE TABLE IF NOT EXISTS camel_messageprocessed (
    processorName VARCHAR(255) NOT NULL,
    messageId     VARCHAR(255) NOT NULL,
    createdAt     TIMESTAMP    NOT NULL DEFAULT NOW(),
    PRIMARY KEY (processorName, messageId)
);
```

The composite primary key `(processorName, messageId)` ensures that each processor instance has its own namespace.
Multiple routes using the same table can each track their own set of processed IDs independently by choosing different processor names.

## Test infrastructure

The tests need Kafka for message transport and PostgreSQL for the JDBC idempotent repository.
A Docker Compose stack provisions both services:

```yaml
services:
  kafka:
    image: docker.io/apache/kafka:latest
    ports:
      - "9092:9092"
    # ... KRaft configuration ...

  kafka-ui:
    image: docker.io/provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    depends_on:
      kafka:
        condition: service_healthy

  postgres:
    image: docker.io/library/postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=eipdb
      - POSTGRES_USER=eipuser
      - POSTGRES_PASSWORD=eippass
    volumes:
      - ./postgres/init-schemas.sql:/docker-entrypoint-initdb.d/01-init-schemas.sql:Z
```

The PostgreSQL container mounts an init script that creates the `camel_messageprocessed` table automatically at startup.
A `BeforeSuite` action starts the compose stack and waits for the Kafka UI to become healthy.

### Quarkus infrastructure setup

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

### Spring Boot infrastructure setup

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

The `waitFor().http()` call blocks until the Kafka UI is reachable on port 8090.
Since the UI depends on Kafka and all containers share the same compose lifecycle, this is a reliable proxy for the entire stack being ready.

## Shared test utilities

Both runtime variants use a shared `EipTestSupport` interface that provides reusable helper methods.

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

    default TestActionBuilder<?> verifyCompletedExchanges(
            String routeId, long count, CamelContext camelContext) {
        return repeatOnError()
                .until((i, context) -> i > 20)
                .autoSleep(Duration.ofSeconds(1))
                .actions(
                    camel()
                        .camelContext(camelContext)
                        .route()
                        .verifyRouteStats(routeId)
                        .completed(count)
                );
    }
}
```

`waitForCamelRouteStarted` uses Camel's Control Bus to poll the route status every second, up to 20 retries.
This ensures the idempotent receiver route is fully started — including its Kafka consumer group registration and JDBC repository initialization — before the test sends any messages.

`verifyCompletedExchanges` uses Citrus's built-in Camel route statistics verification to check how many exchanges a route has completed.
For the idempotent receiver, this is particularly useful: the route receives *both* messages (the unique one and the duplicate), but only forwards the unique one.
The route statistics verification confirms that both exchanges were consumed from Kafka and processed by the route — even though only one produced output on the downstream topic.

## The idempotent receiver tests

The test methods cover the pattern's two complementary behaviors: deduplication of identical messages, and pass-through of unique messages.

### Deduplicating identical orders

This is the primary test — it proves that the idempotent receiver processes the first unique order and correctly drops following duplicates.

#### Quarkus

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
    class IdempotentReceiverTest {

        @Test
        public void shouldDeduplicateIdenticalOrders() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("amount", 100)
            );

            t.given(waitForCamelRouteStarted("idempotent-receiver", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.placed")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.when(
                send()
                    .endpoint("kafka:eip.orders.placed")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.deduplicated?consumerGroup=citrus-dedup-group")
                    .message()
                    .body(Resources.create("templates/order.json"))
            );

            t.then(
                expectTimeout()
                    .endpoint("kafka:eip.orders.deduplicated?consumerGroup=citrus-dedup-group")
                    .timeout(5000)
            );

            t.then(verifyCompletedExchanges("idempotent-receiver", 2, camelContext));
        }
    }
}
```

This test has five distinct phases, each building on the previous one.

**Given — set up variables and wait for the route.**
The test generates a random 4-digit `id` used to construct a unique `order_id`, `customer_id`, and `item_sku` via the shared order template.
It then waits for the `idempotent-receiver` route to reach `Started` status, ensuring the Kafka consumer group is registered and the JDBC repository is connected.

**When — send the same order twice.**
Two identical messages are sent to `kafka:eip.orders.placed`, both using the same `id` variable.
Since the order template interpolates `${id}` into the `order_id` field, both messages carry the same `order_id`.
The Kafka message key is also set to `${id}` — this is good practice for partitioning, ensuring that both copies land on the same partition and are processed in order.

**Then (first assertion) — verify the first message arrives.**
The test receives one message from `kafka:eip.orders.deduplicated`.
The expected body matches the order template — confirming that the unique order was forwarded correctly.

**Then (second assertion) — verify the duplicate was dropped.**
This is the key assertion.
`expectTimeout()` attempts to consume another message from the same Kafka topic and asserts that *nothing* arrives within 5 seconds.
If the idempotent receiver failed to drop the duplicate, a second message would appear on `eip.orders.deduplicated` and this assertion would fail.
Proving a negative — that something did *not* happen — requires a timeout-based approach, and Citrus's `expectTimeout()` provides exactly this.

**Then (third assertion) — verify both exchanges were processed.**
`verifyCompletedExchanges("idempotent-receiver", 2, camelContext)` confirms that the route consumed and processed *both* messages from Kafka.
This is an important distinction: the route processed two exchanges, but the `idempotentConsumer()` EIP filtered one of them before the downstream `to()` step.
The exchange count of 2 proves that the duplicate was not lost or rejected at the Kafka level — it was consumed and deliberately skipped by the idempotent logic.

#### Spring Boot

```java
@SpringBootTest(classes = EndpointsApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class IdempotentReceiverTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldDeduplicateIdenticalOrders() {
            // Test logic is identical to the Quarkus variant
            // ...
        }
    }
}
```

The test logic is identical.
The differences are in the class-level annotations and dependency injection:

| Concern                | Quarkus                                | Spring Boot                                |
|------------------------|----------------------------------------|--------------------------------------------|
| Test bootstrap         | `@QuarkusTest`                         | `@SpringBootTest` + `@CamelSpringBootTest` |
| Citrus integration     | `@CitrusSupport`                       | `@CitrusSpringSupport`                     |
| CamelContext injection | `@Inject` + `@BindToRegistry`          | `@Autowired`                               |
| TestCaseRunner scope   | Class-level field                      | Nested class field with `@CitrusResource`  |
| Infrastructure config  | `@CitrusConfiguration` auto-discovered | `@ContextConfiguration` explicit           |

## The order template

The tests use a shared JSON template that Citrus resolves at runtime:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-00${id}",
  "item_sku": "SKU-SHIP-${id}",
  "quantity": 1,
  "amount": ${amount}
}
```

The `${id}` and `${amount}` placeholders are replaced by Citrus variables before sending.
When used in a `receive()` assertion, the same template serves as the expected body — Citrus validates each field against the received message.
This dual use of templates as both input and expectation is a Citrus convenience that keeps tests concise and eliminates discrepancies between sent and expected data.

## Why idempotent receiver tests are different

Testing an idempotent receiver introduces challenges that most other EIP tests do not have.

**You must prove a negative.**
Most pattern tests verify that something *happened*: a message was routed, split, filtered, or aggregated.
An idempotent receiver test must also verify that something did *not* happen — that the duplicate message did not produce output.
Citrus's `expectTimeout()` is purpose-built for this: it attempts to consume from an endpoint and fails if a message *does* arrive within the timeout window.

**The route processes more exchanges than it produces output for.**
In a content-based router or splitter, every input exchange produces at least one output.
The idempotent receiver deliberately drops exchanges — but they are still *processed* by the route.
Citrus's route statistics verification provides visibility into this: `verifyCompletedExchanges` confirms that both messages were consumed and processed, even though only one was forwarded downstream.

**Consumer groups must be isolated per test.**
Because Kafka consumer groups track offsets, two tests sharing the same group can interfere with each other.
One test's `receive()` call might consume a message intended for the other test.
Assigning a unique consumer group per test method — `citrus-dedup-group` for the deduplication test eliminates this risk.

**The deduplication store is stateful across the test.**
The JDBC repository persists `order_id` values in PostgreSQL.
If the deduplication test runs before the pass-through test with the same `order_id`, the pass-through test would fail because the ID is already in the repository.
The random `id` variable generated by `citrus:randomNumber(4)` ensures each test run uses a unique `order_id`, avoiding cross-test contamination.

## Key takeaways

- **Idempotent receivers prevent duplicate processing.** With at-least-once delivery, duplicates are inevitable. The idempotent receiver pattern tracks processed message IDs and silently skips messages it has already seen — essential for payment processing, inventory updates, and any operation with side effects.
- **`expectTimeout()` proves that duplicates are dropped.** Testing a negative — that a message was *not* forwarded — requires a timeout-based assertion. Citrus provides `expectTimeout()` for exactly this purpose, failing the test only if a message unexpectedly appears.
- **`verifyCompletedExchanges` distinguishes consumption from forwarding.** The route consumes both messages from Kafka but forwards only one. Citrus's route statistics verification reveals the true exchange count, confirming that the duplicate was consumed and deliberately filtered — not lost.
- **Isolated consumer groups prevent test interference.** Each test method should use its own Kafka consumer group. Shared groups cause unpredictable offset tracking, where one test's `receive()` call may consume messages intended for another.
- **Random IDs prevent cross-test contamination.** The JDBC idempotent repository is persistent — a duplicate ID from a previous test run would cause false deduplication. Citrus's `citrus:randomNumber()` function generates unique IDs per test execution.
- **Two runtimes, one test pattern.** Whether you run on Quarkus or Spring Boot, the test structure — send the same message twice, verify one output, confirm no second output, check exchange counts — stays the same. Only the bootstrap annotations and dependency injection differ.
