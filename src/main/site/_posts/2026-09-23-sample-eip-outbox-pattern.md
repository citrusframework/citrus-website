---
layout: sample
title: Testing the Outbox Pattern with Citrus
name: outbox-pattern
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Transactional Outbox Pattern in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/outbox-pattern/
---

Some of the most dangerous bugs in distributed systems live at the boundary between a database and a message broker.
Consider a payment service that must do two things when it processes an order: insert a payment record into PostgreSQL and publish a `PaymentProcessed` event to Kafka.
If the database insert succeeds but the Kafka publish fails, the payment is recorded but no downstream service ever learns about it — the order sits in limbo.
If the Kafka publish succeeds but the application crashes before the database commit, an event announces a payment that never actually happened.

You need both operations to succeed or both to fail.
True distributed transactions (XA/two-phase commit) would solve this, but Kafka does not support XA, and even databases that do pay a steep performance penalty.
The practical solution is the **Outbox Pattern**: write both the business record and the event into the *same* database transaction, then publish the event to Kafka asynchronously from the outbox table.

The [Transactional Client](https://www.enterpriseintegrationpatterns.com/patterns/messaging/TransactionalClient.html) pattern described by Hohpe and Woolf captures the broader concept — making messaging operations part of a transaction.
The outbox pattern is its most widely adopted implementation.

[Apache Camel](https://camel.apache.org) models this with two cooperating routes: a transactional route that writes the payment and the outbox event atomically, and a polling route that reads unpublished events from the outbox table and publishes them to Kafka.
The `transacted()` DSL ensures the two SQL inserts share a single database transaction, while the polling route's `onConsume` query marks each event as published after it has been successfully sent.

Testing the outbox pattern end-to-end is where the real challenge lies.
A unit test can verify that the SQL statements are syntactically correct, but only an integration test can prove that the payment record and the outbox event are committed atomically, that the polling route picks up the event and publishes it to Kafka, and that the outbox row is marked as published afterward.
This is a three-phase verification across two infrastructure systems — exactly the kind of scenario where [Citrus](https://citrusframework.org) shines.

## The scenario

A payment service consumes order events from a Kafka topic `eip.payments.required`.
For each order, the service must:

1. Insert a payment record into the `payments.payments` table with status `PROCESSED`.
2. Write an outbox event into the `payments.outbox` table with the payment details and a unique `event_id`.
3. Both writes happen in a single database transaction — if either fails, both are rolled back.

A separate polling route runs every 5 seconds, reading unpublished rows from `payments.outbox`, publishing each event's payload to `kafka:eip.payments.processed`, and marking the row as `published = true`.

The test must verify all three phases: the payment record exists, the Kafka message was published with the correct payload, and the outbox row was marked as published.

## The Camel routes

The outbox pattern is implemented as two routes in a single `RouteBuilder`.
The first route handles the transactional write; the second handles the asynchronous publish.

### Quarkus

```java
@ApplicationScoped
public class OutboxPatternRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.payments.required?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=payment-outbox")
            .routeId("transactional-client")
            .unmarshal().json()
            .log("Transactional Client — processing payment for order ${body[order_id]}")
            .transacted()
            .to("sql:INSERT INTO payments.payments (order_id, amount, status) "
                + "VALUES (:#${body[order_id]}, :#${body[amount]}, 'PROCESSED')")
            .process(exchange -> {
                Map<String, Object> order = exchange.getIn().getBody(Map.class);
                Map<String, Object> event = new LinkedHashMap<>();
                String eventId = UUID.randomUUID().toString();
                event.put("event_id", eventId);
                event.put("event_type", "PaymentProcessed");
                event.put("aggregate_id", String.valueOf(order.get("order_id")));
                Map<String, Object> payload = new LinkedHashMap<>();
                payload.put("order_id", order.get("order_id"));
                payload.put("amount", order.get("amount"));
                payload.put("status", "PROCESSED");
                event.put("payload", new ObjectMapper().writeValueAsString(payload));
                exchange.getIn().setBody(event);
            })
            .to("sql:INSERT INTO payments.outbox (event_id, event_type, aggregate_id, payload) "
                + "VALUES (:#${body[event_id]}, :#${body[event_type]}, "
                + ":#${body[aggregate_id]}, :#${body[payload]})")
            .log("Transactional Client — payment and outbox event committed");

        from("sql:SELECT * FROM payments.outbox WHERE published = false "
                + "ORDER BY created_at LIMIT 100"
                + "?delay=5000"
                + "&onConsume=UPDATE payments.outbox SET published = true "
                + "WHERE event_id = :#event_id")
            .routeId("outbox-publisher")
            .log("Outbox Publisher — publishing event ${body[event_id]} to Kafka")
            .setBody(simple("${body[payload]}"))
            .to("kafka:eip.payments.processed?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

The first route — `transactional-client` — does the heavy lifting.
It consumes an order from Kafka, deserializes the JSON body, and enters a transaction with `.transacted()`.
Inside the transaction, two SQL inserts execute in sequence: one writes the payment record to `payments.payments`, and a processor builds the outbox event and writes it to `payments.outbox`.
Because both inserts share the same transaction, either both are committed or both are rolled back — the database guarantees atomicity.

The processor between the two inserts constructs the outbox event as a `Map`.
It generates a random UUID as the `event_id`, sets the `event_type` to `PaymentProcessed`, and serializes the payment payload as a JSON string.
The `aggregate_id` is set to the `order_id`, which downstream consumers can use for partitioning or correlation.

The second route — `outbox-publisher` — is a SQL polling consumer.
Every 5 seconds, it selects up to 100 unpublished rows from `payments.outbox` ordered by creation time.
For each row, it extracts the `payload` field, sends it to `kafka:eip.payments.processed`, and the `onConsume` query marks the row as `published = true`.
The `ORDER BY created_at` preserves the temporal ordering of business events, and the `LIMIT 100` provides backpressure during bursts.

### Spring Boot

```java
@Component
public class OutboxPatternRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.payments.required?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=payment-outbox")
            .routeId("transactional-client")
            .unmarshal().json()
            .log("Transactional Client — processing payment for order ${body[order_id]}")
            .transacted()
            .to("sql:INSERT INTO payments.payments (order_id, amount, status) "
                + "VALUES (:#${body[order_id]}, :#${body[amount]}, 'PROCESSED')")
            .process(exchange -> {
                Map<String, Object> order = exchange.getIn().getBody(Map.class);
                Map<String, Object> event = new LinkedHashMap<>();
                String eventId = UUID.randomUUID().toString();
                event.put("event_id", eventId);
                event.put("event_type", "PaymentProcessed");
                event.put("aggregate_id", String.valueOf(order.get("order_id")));
                Map<String, Object> payload = new LinkedHashMap<>();
                payload.put("order_id", order.get("order_id"));
                payload.put("amount", order.get("amount"));
                payload.put("status", "PROCESSED");
                event.put("payload", new ObjectMapper().writeValueAsString(payload));
                exchange.getIn().setBody(event);
            })
            .to("sql:INSERT INTO payments.outbox (event_id, event_type, aggregate_id, payload) "
                + "VALUES (:#${body[event_id]}, :#${body[event_type]}, "
                + ":#${body[aggregate_id]}, :#${body[payload]})")
            .log("Transactional Client — payment and outbox event committed");

        from("sql:SELECT * FROM payments.outbox WHERE published = false "
                + "ORDER BY created_at LIMIT 100"
                + "?delay=5000"
                + "&onConsume=UPDATE payments.outbox SET published = true "
                + "WHERE event_id = :#event_id")
            .routeId("outbox-publisher")
            .log("Outbox Publisher — publishing event ${body[event_id]} to Kafka")
            .setBody(simple("${body[payload]}"))
            .to("kafka:eip.payments.processed?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

`@Component` replaces `@ApplicationScoped` — the route logic is identical between runtimes.

### The outbox guarantees

The two-route design provides three properties that a single route with a direct Kafka write cannot:

| Property                 | How it works                                                                                                                            |
|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| **Atomicity**            | The payment record and the outbox event are written in the same database transaction. Either both exist or neither does.                |
| **Eventual consistency** | The outbox publisher polls the table and publishes to Kafka. If it crashes, it retries on restart — events are published at-least-once. |
| **Ordering**             | Events are published in `created_at` order, preserving the temporal ordering of business operations.                                    |

Combined with an idempotent receiver on the consumer side, the outbox pattern achieves effectively-once processing across the database-to-Kafka boundary.

## The database schema

The outbox pattern requires two tables — one for the business data, one for the outbox events.
Both are created by an init script mounted into the PostgreSQL container:

```sql
CREATE SCHEMA IF NOT EXISTS payments;

CREATE TABLE payments.payments (
    id         SERIAL PRIMARY KEY,
    order_id   VARCHAR(64)    NOT NULL,
    amount     DECIMAL(12,2)  NOT NULL,
    status     VARCHAR(20)    NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP      NOT NULL DEFAULT NOW()
);

CREATE TABLE payments.outbox (
    event_id     VARCHAR(255) PRIMARY KEY,
    event_type   VARCHAR(64)  NOT NULL,
    aggregate_id VARCHAR(64)  NOT NULL,
    payload      TEXT         NOT NULL,
    published    BOOLEAN      NOT NULL DEFAULT false,
    created_at   TIMESTAMP    NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_unpublished
    ON payments.outbox (created_at)
    WHERE published = false;
```

The `payments.payments` table stores the business record.
The `payments.outbox` table stores the events waiting to be published.
The `published` column defaults to `false` — the outbox publisher's SELECT query filters on this column, and its `onConsume` UPDATE flips it to `true` after successful publication.

The partial index `idx_outbox_unpublished` is a performance optimization: it covers only rows with `published = false`, so the polling query's execution plan remains efficient even as the table grows with millions of published rows.

## Test infrastructure

The tests require Kafka for message transport and PostgreSQL for the transactional writes and the outbox table.
A Docker Compose stack provisions both:

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

The PostgreSQL container mounts the init script that creates the `payments` schema and both tables at startup.
A `BeforeSuite` action starts the compose stack and waits for readiness; an `AfterSuite` tears it down.

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
Since the UI depends on Kafka and all containers share the same compose lifecycle, this is a reliable proxy for the entire stack being ready — including PostgreSQL, which starts faster than Kafka.

## Shared test utilities

Both runtimes use a shared `EipTestSupport` interface that provides reusable helper methods for waiting on route startup and verifying exchange counts.

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

`waitForCamelRouteStarted` uses Camel's Control Bus to poll the route status every second, up to 20 retries.
The outbox test waits for *two* routes — `transactional-client` and `outbox-publisher` — because both must be running before the test sends a payment request.
If the transactional route has not started, the Kafka message sits unprocessed.
If the outbox publisher has not started, the event sits in the outbox table and never reaches Kafka.

## The outbox pattern test

The test exercises the full pipeline: send a payment request to Kafka, verify the payment record in PostgreSQL, verify the event on the downstream Kafka topic, and confirm the outbox row was marked as published.

### Quarkus

```java
@QuarkusTest
@CitrusSupport
class EipTests implements EipTestSupport {

    @CitrusResource
    TestCaseRunner t;

    @Inject
    @BindToRegistry
    DataSource dataSource;

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    @Nested
    class TransactionalOutboxTest {

        @Test
        public void shouldProcessPaymentAndPublishViaOutbox() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("amount", 500)
            );

            t.given(waitForCamelRouteStarted("transactional-client", camelContext));
            t.given(waitForCamelRouteStarted("outbox-publisher", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.payments.required")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                repeatOnError()
                    .until((i, context) -> i > 15)
                    .autoSleep(Duration.ofSeconds(1))
                    .actions(
                        sql(dataSource)
                            .query()
                            .statement("SELECT order_id, status FROM payments.payments "
                                + "WHERE order_id = '${id}'")
                            .validate("order_id", "${id}")
                            .validate("status", "PROCESSED")
                    )
            );

            t.then(
                repeatOnError()
                    .until((i, context) -> i > 20)
                    .autoSleep(Duration.ofSeconds(2))
                    .actions(
                        receive()
                            .endpoint("kafka:eip.payments.processed"
                                + "?consumerGroup=citrus-payments-group")
                            .message()
                            .body("""
                            {
                              "order_id": ${id},
                              "amount": ${amount},
                              "status": "PROCESSED"
                            }
                            """)
                    )
            );

            t.then(
                repeatOnError()
                    .until((i, context) -> i > 15)
                    .autoSleep(Duration.ofSeconds(1))
                    .actions(
                        sql(dataSource)
                            .query()
                            .statement("SELECT published FROM payments.outbox "
                                + "WHERE aggregate_id = '${id}'")
                            .validate("published", "true")
                    )
            );
        }
    }
}
```

This test has four distinct phases, each verifying a different part of the outbox pipeline.

**Given — set up variables and wait for both routes.**
The test generates a random 4-digit `id` used as the `order_id` and sets the `amount` to 500.
It then waits for *both* the `transactional-client` and `outbox-publisher` routes to reach `Started` status.
Waiting for both is critical — the transactional route must be ready to consume from Kafka and write to the database, and the outbox publisher must be polling the outbox table before the test expects the event to appear on the downstream Kafka topic.

**When — send a payment request.**
A single order message goes to `kafka:eip.payments.required` using the shared order template.
The template interpolates `${id}` into the `order_id` field and `${amount}` into the `amount` field.
This is the test's stimulus — a payment request that triggers the entire outbox pipeline.

**Then (first assertion) — verify the payment record in PostgreSQL.**
The test queries `payments.payments` to confirm the payment record was written.
The `repeatOnError()` block retries up to 15 times with 1-second intervals, accommodating the time the transactional route needs to consume the Kafka message and execute the database transaction.

```java
sql(dataSource)
    .query()
    .statement("SELECT order_id, status FROM payments.payments WHERE order_id = '${id}'")
    .validate("order_id", "${id}")
    .validate("status", "PROCESSED")
```

This assertion verifies two things: the row exists (the query returns a result) and the status is `PROCESSED` (not the default `PENDING`).
If the transaction rolled back, the row would not exist at all.
If the insert succeeded but with wrong data, the validation would fail.

**Then (second assertion) — verify the Kafka event.**
The test receives a message from `kafka:eip.payments.processed` — the topic that the outbox publisher writes to.
This assertion has a longer retry window (20 retries with 2-second intervals) because it depends on two asynchronous steps: the transactional route writing the outbox event, and the outbox publisher polling the table and publishing to Kafka.
The publisher polls every 5 seconds, so the event may take up to 5 seconds to appear after the transaction commits.

```java
receive()
    .endpoint("kafka:eip.payments.processed?consumerGroup=citrus-payments-group")
    .message()
    .body("""
    {
      "order_id": ${id},
      "amount": ${amount},
      "status": "PROCESSED"
    }
    """)
```

The expected body is an inline JSON template.
Citrus validates each field against the received message.
The `order_id` and `amount` must match the values sent in the stimulus, and the `status` must be `PROCESSED` — confirming that the outbox event's payload was correctly constructed by the processor and faithfully published by the outbox route.

**Then (third assertion) — verify the outbox row was marked as published.**
The final assertion queries the `payments.outbox` table to confirm the row was updated.

```java
sql(dataSource)
    .query()
    .statement("SELECT published FROM payments.outbox WHERE aggregate_id = '${id}'")
    .validate("published", "true")
```

This is the assertion that most developers would skip — and the one that matters most.
Without it, you would know the Kafka event was published, but you would not know whether the `onConsume` UPDATE actually ran.
If the `onConsume` query fails silently (a typo in the column name, a constraint violation, a connection timeout), the row remains `published = false` and the outbox publisher would re-publish it on the next poll cycle — producing a duplicate event.

By explicitly verifying `published = true`, the test proves the entire lifecycle: write, publish, and mark-as-published.

### Spring Boot

```java
@SpringBootTest(classes = EndpointsApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Autowired
    DataSource dataSource;

    @Nested
    class TransactionalOutboxTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldProcessPaymentAndPublishViaOutbox() {
            // Test logic is identical to the Quarkus variant
            // ...
        }
    }
}
```

The test logic is identical.
The differences are only in the class-level annotations and dependency injection:

| Concern                | Quarkus                                | Spring Boot                                |
|------------------------|----------------------------------------|--------------------------------------------|
| Test bootstrap         | `@QuarkusTest`                         | `@SpringBootTest` + `@CamelSpringBootTest` |
| Citrus integration     | `@CitrusSupport`                       | `@CitrusSpringSupport`                     |
| DataSource injection   | `@Inject` + `@BindToRegistry`          | `@Autowired`                               |
| CamelContext injection | `@Inject` + `@BindToRegistry`          | `@Autowired`                               |
| TestCaseRunner scope   | Class-level field                      | Nested class field with `@CitrusResource`  |
| Infrastructure config  | `@CitrusConfiguration` auto-discovered | `@ContextConfiguration` explicit           |

## Understanding the retry strategies

The three assertions use different retry configurations, and the differences are deliberate.

The **payment record assertion** retries 15 times at 1-second intervals.
The transactional route consumes the Kafka message, executes two SQL inserts, and commits — a fast operation that typically completes within a few seconds.
A 15-second window is generous but avoids unnecessary waiting.

The **Kafka event assertion** retries 20 times at 2-second intervals.
This is the slowest step because it depends on the outbox publisher's poll cycle.
After the transactional route commits, the outbox row exists but is not yet published.
The publisher polls every 5 seconds (`delay=5000` in the route), so the event may not appear on Kafka until up to 5 seconds after the commit.
The wider retry window (40 seconds total) accounts for this latency plus any Kafka producer buffering.

The **outbox published assertion** retries 15 times at 1-second intervals.
By the time this assertion runs, the Kafka event has already been received — which means the outbox publisher already executed the `onConsume` UPDATE.
The short retry window is a safety net for transactional propagation delays, not a wait for new asynchronous work.

## Why outbox pattern tests are different

Testing the outbox pattern introduces challenges that set it apart from other EIP tests.

**The test verifies three separate state changes across two systems.**
Most EIP tests have a simple shape: send a message, verify the output.
The outbox test must verify a database record (payment), a Kafka message (event), and a database flag (published).
Each verification depends on the previous one succeeding, creating a three-step assertion chain.
Citrus's `sql()` and `receive()` actions can be freely mixed within the same test, making this cross-infrastructure verification natural.

**Timing depends on a polling interval, not event delivery.**
The gap between the transaction commit and the Kafka event is governed by the outbox publisher's 5-second poll delay — not by Kafka's millisecond-scale delivery latency.
The test must wait for a full poll cycle, which is why the Kafka assertion uses a wider retry window than the database assertions.
Using `repeatOnError()` instead of a fixed sleep makes the test both reliable (it waits long enough) and fast (it succeeds as soon as the message appears, not after a worst-case delay).

**The final assertion catches a class of bugs that no other check reveals.**
The `published = true` assertion specifically targets failures in the `onConsume` query.
Without it, a broken `onConsume` (wrong column name, missing WHERE clause, connection pool exhaustion) would go undetected: the event reaches Kafka, the test passes, but the row remains `published = false`.
On the next poll cycle, the publisher re-sends the event — a duplicate that violates the pattern's at-most-once publication guarantee.
This is the kind of subtle, asynchronous bug that only an end-to-end integration test can catch.

**Two routes must be running before the test starts.**
Unlike most EIP tests that wait for a single route, the outbox test must wait for both `transactional-client` and `outbox-publisher`.
Omitting the wait for either route produces different failure modes: a missing transactional route means the Kafka message is never consumed; a missing outbox publisher means the event is written to the database but never published.
The separate `waitForCamelRouteStarted` calls for each route make the test's dependencies explicit.

## Key takeaways

- **The outbox pattern ensures atomicity without distributed transactions.** By writing both the business record and the event to the same database in a single transaction, the pattern avoids the complexity of XA/two-phase commit while guaranteeing that either both writes succeed or both are rolled back.
- **Test all three state changes, not just the Kafka event.** The payment record, the Kafka message, and the outbox `published` flag are all part of the pattern's correctness contract. Skipping the database assertions — especially the `published = true` check — leaves a class of `onConsume` bugs undetected.
- **Match retry strategies to the asynchronous step they are waiting for.** Database assertions after a fast transaction need short retries. Kafka assertions after a polling delay need wider retries. Citrus's `repeatOnError()` makes this explicit: you set the retry count, the sleep interval, and the assertions — no guesswork about timing.
- **Wait for all cooperating routes before sending the stimulus.** The outbox pattern uses two routes that must both be running. Missing a `waitForCamelRouteStarted` call produces intermittent failures that are difficult to diagnose — the test works when the route starts quickly and fails when it does not.
- **Citrus's `sql()` action bridges database and messaging assertions.** The same test that receives a Kafka message can query PostgreSQL to verify side effects. This cross-infrastructure verification is the outbox test's defining characteristic and the reason an integration test framework is essential.
- **Two runtimes, one test pattern.** Whether you run on Quarkus or Spring Boot, the test structure — send a payment request, verify the database record, receive the Kafka event, check the outbox flag — stays the same. Only the bootstrap annotations and dependency injection differ.
