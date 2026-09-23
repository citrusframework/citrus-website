---
layout: sample
title: Testing the Polling Consumer Pattern with Citrus
name: polling-consumer
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Polling Consumer EIP in Apache Camel with Citrus across Quarkus, Spring Boot and YAML DSL
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/polling-consumer/
---

Most messaging patterns covered so far — content-based routers, splitters, message filters — are event-driven.
A message arrives, the route processes it, and an output appears almost immediately.
But not every data source can push messages to your application.
Files land in a directory at unpredictable intervals.
Database tables receive new rows from legacy systems that have no event notification mechanism.
FTP servers accumulate uploads silently.
In all these cases, the only way to detect new data is to *ask* for it — repeatedly, on a schedule.

The [Polling Consumer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PollingConsumer.html) pattern, described by Hohpe and Woolf, addresses this.
A polling consumer actively checks a channel for new messages at regular intervals.
It initiates the receive operation — the messaging system does not push messages to it.
This is the "pull" model, and it is the natural fit for sources that cannot push: file systems, databases, FTP servers, and scheduled batch jobs.

[Apache Camel](https://camel.apache.org) implements the polling consumer in two ways.
Many components — `file`, `ftp`, `sql`, `timer` — are polling consumers by nature: the `from()` endpoint polls its source on a configurable schedule.
For event-driven routes that occasionally need to pull a message on demand, Camel provides `pollEnrich()`, which combines a timer trigger with an explicit poll from a second endpoint.

Testing a polling consumer introduces a timing challenge that event-driven patterns do not have.
The test cannot simply send a message and expect an immediate response.
It must account for the poll interval: the message might sit in the source for seconds before the consumer picks it up.
Integration tests with [Citrus](https://citrusframework.org) handle this gracefully — retry-based assertions naturally accommodate the poll delay without resorting to brittle fixed sleeps.

This post covers two variants of the polling consumer: a timer-driven poll from Kafka using `pollEnrich()`, and a SQL polling consumer that reads rows from PostgreSQL and publishes them to Kafka.
The SQL variant is the more realistic and interesting case — it demonstrates database-to-messaging integration with automatic row status updates, and the Citrus test exercises both the database and Kafka sides of the pipeline.

## The scenario

### Timer-based polling consumer

A timer fires every 10 seconds.
On each tick, the route uses `pollEnrich()` to check a Kafka topic for a waiting message.
If a message is available, it is consumed and logged.
If no message arrives within a 5-second timeout, the route logs that the poll cycle was empty.
This is the simplest form of polling consumer — useful when you want explicit control over *when* messages are consumed, rather than letting the Kafka client manage the fetch loop.

### SQL polling consumer

A more common real-world scenario: a database table receives new order rows from a legacy system.
There is no event notification — the Camel route polls the `orders.orders` table every 30 seconds, looking for rows with `status = 'PLACED'`.
Each row is read, marshalled to JSON, and published to a Kafka topic.
After successful processing, the `onConsume` query updates the row's status to `'PROCESSING'`, preventing it from being picked up again on the next poll.

This two-phase behavior — read then update — is exactly what makes the SQL polling consumer worth testing end-to-end.
A unit test could verify the SQL query syntax, but only an integration test proves that the row is actually read, the Kafka message is published, and the status update takes effect.

## The Camel routes

### Timer-based polling consumer

#### Quarkus

```java
@ApplicationScoped
public class PollingConsumerRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("timer:poll-trigger?period=10000&delay=5000")
            .routeId("polling-consumer")
            .log("Polling consumer triggered — checking for messages …")
            .pollEnrich("kafka:eip.consumer.poll?brokers={% raw %}{{kafka.brokers}}{% endraw %}"
                + "&groupId=polling-consumer&autoOffsetReset=earliest", 5000)
            .choice()
                .when(body().isNull())
                    .log("No message available during this poll cycle")
                .otherwise()
                    .unmarshal().json()
                    .log("Polled message: order ${body[order_id]}, type=${body[event_type]}")
            .end();
    }
}
```

The route starts from a `timer`, not from Kafka directly.
Every 10 seconds, the timer fires and `pollEnrich()` attempts to pull one message from the `eip.consumer.poll` topic with a 5-second timeout.
The `choice()` block handles the two outcomes: a null body means no message was available; otherwise, the message is deserialized and logged.

This is fundamentally different from a standard `from("kafka:...")` consumer.
A Kafka `from()` consumer is event-driven — Camel manages the poll loop internally and your route logic runs whenever messages arrive.
With `pollEnrich()`, *you* control the timing: the poll happens exactly when the timer fires, and at most one message is consumed per cycle.

#### Spring Boot

```java
@Component
public class PollingConsumerRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("timer:poll-trigger?period=10000&delay=5000")
            .routeId("polling-consumer")
            .log("Polling consumer triggered — checking for messages …")
            .pollEnrich("kafka:eip.consumer.poll?brokers={% raw %}{{kafka.brokers}}{% endraw %}"
                + "&groupId=polling-consumer&autoOffsetReset=earliest", 5000)
            .choice()
                .when(body().isNull())
                    .log("No message available during this poll cycle")
                .otherwise()
                    .unmarshal().json()
                    .log("Polled message: order ${body[order_id]}, type=${body[event_type]}")
            .end();
    }
}
```

`@Component` replaces `@ApplicationScoped` — the route logic is identical.

### SQL polling consumer

The SQL polling consumer is more interesting because it bridges two infrastructure systems: PostgreSQL as the source and Kafka as the destination.

#### Quarkus

```java
@ApplicationScoped
public class SqlPollingConsumerRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("sql:SELECT * FROM orders.orders WHERE status = 'PLACED' "
                + "ORDER BY created_at LIMIT 10"
                + "?delay=30000"
                + "&onConsume=UPDATE orders.orders SET status = 'PROCESSING' WHERE id = :#id")
            .routeId("polling-consumer-sql")
            .log("SQL Polling Consumer — processing order from DB: "
                + "id=${body[id]}, customer=${body[customer_id]}, sku=${body[item_sku]}")
            .marshal().json()
            .to("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

The `from("sql:...")` component is a natural polling consumer.
Every 30 seconds (controlled by `delay=30000`), Camel executes the SELECT query and creates one exchange per row.
The query selects at most 10 rows ordered by creation time — this is the backpressure mechanism, preventing the consumer from overwhelming the downstream Kafka topic during a burst of new orders.

The `onConsume` parameter is the key to reliable processing.
After each row is *successfully* processed (marshalled to JSON and published to Kafka), Camel executes the UPDATE statement, setting the row's status from `'PLACED'` to `'PROCESSING'`.
The `:#id` syntax is a named parameter that Camel resolves from the current exchange body — it refers to the `id` column of the row being processed.
On the next poll cycle, the WHERE clause `status = 'PLACED'` naturally excludes already-processed rows.

#### Spring Boot

```java
@Component
public class SqlPollingConsumerRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("sql:SELECT * FROM orders.orders WHERE status = 'PLACED' "
                + "ORDER BY created_at LIMIT 10"
                + "?delay=30000"
                + "&onConsume=UPDATE orders.orders SET status = 'PROCESSING' WHERE id = :#id")
            .routeId("polling-consumer-sql")
            .log("SQL Polling Consumer — processing order from DB: "
                + "id=${body[id]}, customer=${body[customer_id]}, sku=${body[item_sku]}")
            .marshal().json()
            .to("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

#### YAML DSL

```yaml
- route:
    id: polling-consumer-sql
    from:
      uri: "sql:SELECT * FROM orders.orders WHERE status = 'PLACED' ORDER BY created_at LIMIT 10"
      parameters:
        delay: 30000
        onConsume: "UPDATE orders.orders SET status = 'PROCESSING' WHERE id = :#id"
      steps:
        - log: "SQL Polling Consumer — processing order from DB: id=${body[id]}, customer=${body[customer_id]}, sku=${body[item_sku]}"
        - marshal:
            json:
              library: Jackson
        - to:
            uri: "kafka:eip.orders.placed"
            parameters:
              brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
```

### Polling parameters

Camel's polling consumers share a common set of scheduling parameters:

| Parameter                  | Description                                   | Default       |
|----------------------------|-----------------------------------------------|---------------|
| `delay`                    | Milliseconds between polls                    | 500           |
| `maxMessagesPerPoll`       | Maximum messages to process per poll cycle    | 0 (unlimited) |
| `greedy`                   | Poll again immediately if messages were found | false         |
| `sendEmptyMessageWhenIdle` | Send an empty exchange if no messages found   | false         |

The `greedy` option is particularly useful for database polling.
With `greedy=true` and a high `delay`, the consumer polls rapidly when rows are available but backs off to the long interval when the table is empty.
This avoids the wasted CPU and database connections that come from aggressive short-interval polling on an empty table.

## Test infrastructure

Both the timer-based and SQL polling consumers require Kafka.
The SQL variant additionally needs PostgreSQL with a pre-initialized schema.
The Docker Compose stack provisions both:

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
      - POSTGRES_DB=eip
      - POSTGRES_USER=eip
      - POSTGRES_PASSWORD=eip
    volumes:
      - ./postgres/init-schemas.sql:/docker-entrypoint-initdb.d/01-init-schemas.sql:Z
```

The PostgreSQL container mounts an init script that creates the `orders.orders` table:

```sql
CREATE SCHEMA IF NOT EXISTS orders;

CREATE TABLE orders.orders (
    id          SERIAL PRIMARY KEY,
    customer_id VARCHAR(64)    NOT NULL,
    item_sku    VARCHAR(64)    NOT NULL,
    quantity    INTEGER        NOT NULL,
    amount      DECIMAL(12,2)  NOT NULL,
    status      VARCHAR(20)    NOT NULL DEFAULT 'PLACED',
    created_at  TIMESTAMP      NOT NULL DEFAULT NOW()
);
```

The `status` column defaults to `'PLACED'` — exactly what the SQL polling consumer's WHERE clause looks for.
New rows inserted during the test are immediately eligible for polling.

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

The `waitFor().http()` blocks until the Kafka UI is reachable on port 8090 — a reliable proxy for both Kafka and PostgreSQL being ready, since the UI depends on Kafka and all containers share the same compose lifecycle.

## Shared test utilities

Both runtimes implement the `EipTestSupport` interface, which provides two reusable methods.

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

    default TestActionBuilder<?> assertProcessedExchanges(
            String routeId, Predicate<Long> check, CamelContext camelContext) {
        return repeatOnError()
                .until((i, context) -> i > 20)
                .autoSleep(Duration.ofSeconds(1))
                .actions(
                    context -> {
                        ManagedCamelContext managedContext = camelContext
                                .getCamelContextExtension()
                                .getContextPlugin(ManagedCamelContext.class);
                        ManagedRouteMBean routeMBean =
                                managedContext.getManagedRoute(routeId);
                        if (routeMBean != null) {
                            long failed = routeMBean.getExchangesFailed();
                            if (failed > 0) {
                                throw new ValidationException(
                                    "Route '%s' has %d failed exchanges"
                                            .formatted(routeId, failed));
                            }
                            long completed = routeMBean.getExchangesCompleted();
                            if (!check.test(completed)) {
                                throw new ValidationException(
                                    "Route '%s' has %d completed exchanges"
                                            .formatted(routeId, completed));
                            }
                        } else {
                            throw new CitrusRuntimeException(
                                "No managed route stats for '%s'"
                                        .formatted(routeId));
                        }
                    }
                );
    }
}
```

`waitForCamelRouteStarted` uses Camel's Control Bus to poll the route status every second, up to 20 retries.
It ensures the polling consumer route is fully started before the test produces any test data.
Without this guard, an order inserted into PostgreSQL might be missed if the SQL consumer has not yet started polling.

`assertProcessedExchanges` uses Camel's JMX management API to check how many exchanges a route has processed.
This is particularly useful for polling consumer tests where there is no explicit output message to receive — the timer-based polling consumer logs the message but does not forward it to another endpoint.
The assertion verifies that the route actually ran and successfully processed exchanges without errors.

## The polling consumer test

The timer-based polling consumer test is straightforward: send a message to Kafka, then wait for the poll cycle to pick it up.

### Quarkus test

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
    class PollingConsumerTest {

        @Test
        public void shouldPollMessageFromKafka() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("eventType", "order_placed")
                    .variable("amount", 99)
            );

            t.given(waitForCamelRouteStarted("polling-consumer", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.consumer.poll")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                assertProcessedExchanges("polling-consumer", it -> it > 1, camelContext)
            );
        }
    }
}
```

**Given — set up variables and wait for the route.**
The test generates a random order ID and sets up the event type and amount for the order template.
It then waits for the `polling-consumer` route to reach `Started` status — important because the timer-driven route needs to be active and the `pollEnrich()` Kafka consumer group needs to be registered before a message can be consumed.

**When — send an order to the polled topic.**
A single order message goes to `kafka:eip.consumer.poll` — the topic that the `pollEnrich()` call reads from.

**Then — assert the route processed exchanges.**
The test does *not* try to receive the message from another Kafka topic, because the polling consumer route only logs the message — it does not forward it anywhere.
Instead, it uses `assertProcessedExchanges` with a predicate `it -> it > 1`.
The predicate checks that more than one exchange has completed on the `polling-consumer` route.
Why more than one?
Because the timer fires repeatedly, producing exchanges even when no message is available (the "no message" branch of the `choice()`).
The assertion retries until the route has processed at least one exchange *with* the polled message.

This is a key insight for testing polling consumers: when the consumer only logs or internally processes the message without producing output on a verifiable endpoint, Camel's management API provides an alternative verification path.

### Spring Boot test

```java
@SpringBootTest(classes = ConsumerPatternsApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class PollingConsumerTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldPollMessageFromKafka() {
            // Test logic is identical to the Quarkus variant
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("eventType", "order_placed")
                    .variable("amount", 99)
            );

            t.given(waitForCamelRouteStarted("polling-consumer", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.consumer.poll")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                assertProcessedExchanges("polling-consumer", it -> it > 1, camelContext)
            );
        }
    }
}
```

## The SQL polling consumer test

The SQL polling consumer test is richer than the timer-based variant because it exercises two infrastructure systems and verifies three things: a database row is consumed, a Kafka message is produced, and the database row is updated.

### Quarkus test

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
    class SqlPollingConsumerTest {

        @Test
        public void shouldHandleSqlPollingConsumer() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("amount", 40)
            );

            t.given(waitForCamelRouteStarted("polling-consumer-sql", camelContext));

            t.when(
                sql(dataSource)
                    .statement("INSERT INTO orders.orders "
                        + "(customer_id, item_sku, quantity, amount) "
                        + "VALUES ('CUST-00${id}', 'SKU-${id}', '1', '${amount}')")
            );

            t.then(
                repeatOnError()
                    .until((i, context) -> i > 15)
                    .autoSleep(Duration.ofSeconds(1))
                    .actions(
                        receive()
                            .endpoint("kafka:eip.orders.placed?consumerGroup=citrus-placed-group")
                            .message()
                            .body("""
                            {
                              "id": "@variable(order_id)@",
                              "customer_id": "CUST-00${id}",
                              "status": "PLACED",
                              "amount": ${amount}.0,
                              "item_sku": "SKU-${id}",
                              "quantity": 1,
                              "created_at": "@ignore@"
                            }
                            """)
                    )
            );

            t.then(
                sql(dataSource)
                    .query()
                    .statement("SELECT status FROM orders.orders WHERE id = '${order_id}'")
                    .validate("status", "PROCESSING")
            );
        }
    }
}
```

This test has four distinct phases that exercise the full polling consumer pipeline.

**Given — set up variables and wait for the route.**
The test generates a random ID used to construct a unique `customer_id` and `item_sku`.
It then waits for the `polling-consumer-sql` route to start.
This is critical: if the INSERT happens before the route is polling, the row could sit in the database for an entire 30-second delay interval before being picked up.

**When — insert a test row into PostgreSQL.**
Citrus's `sql()` action inserts a new order row directly into the `orders.orders` table.
The row is created with the default status `'PLACED'`, making it immediately eligible for the SQL consumer's SELECT query.
This is the test's "stimulus" — instead of sending a message to a Kafka topic, the test writes directly to the database that the polling consumer reads from.

**Then (first assertion) — verify the Kafka output.**
The test receives a message from `kafka:eip.orders.placed` — the topic that the SQL polling consumer publishes to.
The `repeatOnError()` block retries up to 15 times with 1-second intervals, giving the poll cycle time to fire and process the row.

The expected body is an inline JSON template that validates the row's data.
Two Citrus features are at work here:

- `@variable(order_id)@` captures the database-generated `id` value into a Citrus variable named `order_id` rather than asserting a specific value. The ID is auto-generated by PostgreSQL's `SERIAL` column, so the test cannot predict it in advance. By extracting it into a variable, the test can reference it in subsequent assertions.
- `@ignore@` skips validation of the `created_at` timestamp, which depends on the database server's clock.

**Then (second assertion) — verify the database status update.**
The final `sql().query()` action reads the row back from PostgreSQL using the captured `${order_id}` and asserts that its status has changed from `'PLACED'` to `'PROCESSING'`.
This verifies the `onConsume` query — the UPDATE that Camel executes after successfully processing each row.

This second assertion is what makes the test truly end-to-end.
Without it, you would know that the row was read and a Kafka message was produced, but you would not know whether the status update actually ran.
A failure in the `onConsume` query would cause the same row to be processed again on the next poll cycle — a subtle bug that only an integration test covering both the read and the update can catch.

### Spring Boot test

```java
@SpringBootTest(classes = ConsumerPatternsApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Autowired
    DataSource dataSource;

    @Nested
    class SqlPollingConsumerTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldHandleSqlPollingConsumer() {
            // Test logic is identical to the Quarkus variant
            // ...
        }
    }
}
```

The test logic is identical.
The differences are only in the test class annotations and dependency injection:

| Concern                | Quarkus                                | Spring Boot                                |
|------------------------|----------------------------------------|--------------------------------------------|
| Test bootstrap         | `@QuarkusTest`                         | `@SpringBootTest` + `@CamelSpringBootTest` |
| Citrus integration     | `@CitrusSupport`                       | `@CitrusSpringSupport`                     |
| DataSource injection   | `@Inject` + `@BindToRegistry`          | `@Autowired`                               |
| CamelContext injection | `@Inject` + `@BindToRegistry`          | `@Autowired`                               |
| TestCaseRunner scope   | Class-level field                      | Nested class field with `@CitrusResource`  |
| Infrastructure config  | `@CitrusConfiguration` auto-discovered | `@ContextConfiguration` explicit           |

## Why polling consumer tests are different

Polling consumer tests have characteristics that set them apart from event-driven pattern tests:

**The poll interval is the test's timing constraint.**
With an event-driven consumer, a message sent to Kafka is processed within milliseconds.
With a polling consumer, the message sits in the source — a Kafka topic, a database table, a file directory — until the next poll cycle fires.
The test must wait at least one full interval.
Citrus's `repeatOnError()` and `timeout` parameters handle this gracefully, retrying until the poll cycle picks up the test data.

**The test produces input in a different medium than the route consumes.**
For the SQL polling consumer, the test does not send a Kafka message — it inserts a database row.
The test and the route operate on different infrastructure: the test writes to PostgreSQL, the route reads from PostgreSQL and writes to Kafka, and the test verifies on Kafka.
This cross-infrastructure flow is a natural fit for integration testing and would be impossible to verify with unit tests alone.

**Side effects are first-class assertions.**
The `onConsume` status update is a side effect that is just as important as the Kafka output.
The test verifies it explicitly with a SQL query after receiving the Kafka message.
If the `onConsume` query fails silently, the row would be reprocessed on the next poll — a production bug that only a test covering both the output and the side effect can detect.

**`assertProcessedExchanges` provides a fallback verification path.**
When a polling consumer only logs messages internally (like the timer-based variant), there is no output endpoint to receive from.
Camel's management API lets the test verify that the route processed exchanges without errors — a useful technique for any route that does not produce verifiable output on an external endpoint.

## Key takeaways

- **Polling consumers are pull-based.** They actively check for new data on a schedule, making them the right choice for sources that cannot push: databases, files, FTP, and scheduled batch jobs. The poll interval and `maxMessagesPerPoll` provide natural backpressure control.
- **`pollEnrich()` turns any route into a polling consumer.** A timer-triggered route with `pollEnrich()` gives you explicit control over *when* messages are consumed — useful when you need to decouple the consumption schedule from the source's availability.
- **SQL polling with `onConsume` is a two-phase operation.** The SELECT reads the row; the `onConsume` UPDATE marks it as processed. Both phases must be verified in the test — the Kafka output proves the read worked, and the SQL assertion proves the update ran.
- **`@variable(name)@` captures dynamic values from the system under test.** Database-generated IDs cannot be predicted by the test. Citrus's variable extraction lets you capture these values from the first assertion and use them in subsequent verifications.
- **`repeatOnError()` accommodates poll timing naturally.** Rather than inserting fixed sleeps matching the poll interval, Citrus retries the assertion until it succeeds or the retry limit is reached — making tests both reliable and as fast as possible.
- **`assertProcessedExchanges` verifies routes without external output.** When a polling consumer only logs or internally processes messages, Camel's management API provides exchange counts and error rates as an alternative verification mechanism.
- **Three runtimes, one test pattern.** Whether you run on Quarkus, Spring Boot, or YAML DSL with Camel JBang, the test structure — seed the source, wait for the poll, verify the output and side effects — stays the same. Only the bootstrap annotations and infrastructure wiring change.
