---
layout: post
title: Database Testing - SQL Polling Consumers and Data Verification
short-title: Database Testing with Citrus
author: Christoph Deppisch
github: christophd
categories: [blog]
---

Many [Apache Camel](https://camel.apache.org) routes don't just shuttle messages between Kafka topics. 
They read from databases, write to databases, and use database state to drive routing decisions. 
SQL Polling Consumers pull new rows from a table on a schedule. 
Transactional Outbox patterns write both the business record and a publication event in a single transaction. 
Enrichment routes look up reference data before forwarding a message.

Testing these flows with a mocked persistence layer is dangerous. 
A mock returns whatever you tell it to — it never catches a broken SQL query, a missing column, or a transaction that commits the payment but fails the outbox insert. 
Real database testing requires inserting test data, waiting for the route to process it, verifying the output message, and then checking that the database state changed correctly.

[Citrus](https://citrusframework.org) provides a `citrus-sql` module with first-class SQL test actions: insert rows, execute queries, validate column values, and capture auto-generated keys — all integrated with the same `given`/`when`/`then` flow used for Kafka and HTTP testing. 
Combined with `citrus-testcontainers` for PostgreSQL infrastructure, you get a complete database testing toolkit that works with real databases, not mocks.

# Setting up database infrastructure

Database tests need a real database. Docker Compose with PostgreSQL is a simple way to get one running for tests.

## The Docker Compose service

Add a PostgreSQL service to your `compose.yaml` under `src/test/resources/_infra/`:

```yaml
  postgres:
    image: docker.io/library/postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=eipdb
      - POSTGRES_USER=eipuser
      - POSTGRES_PASSWORD=eippass
      - POSTGRES_INITDB_ARGS=--encoding=UTF8 --locale=C
    volumes:
      - postgres-data:/var/lib/postgresql/data:Z
      - ./postgres/init-schemas.sql:/docker-entrypoint-initdb.d/01-init-schemas.sql:Z
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U eipuser -d eipdb"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 15s
```

The key detail is the init script mount. The `init-schemas.sql` file is placed in PostgreSQL's `docker-entrypoint-initdb.d/` directory, which means it runs automatically on first startup — creating schemas and tables before any test touches the database.

## The schema initialization script

Place `init-schemas.sql` under `src/test/resources/_infra/postgres/`:

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

The `id` column uses `SERIAL` — PostgreSQL auto-generates it on insert. 
This is important for testing because we need to capture this value from the route's output to use in subsequent database queries.

## DataSource injection

In Quarkus, inject the application's `DataSource` and register it with Citrus:

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

    // ... test methods
}
```

Both annotations are needed: `@Inject` for CDI injection and `@BindToRegistry` so Citrus's `sql()` actions can find the DataSource by name. 
In Spring Boot, the equivalent is `@Autowired DataSource dataSource` — no `@BindToRegistry` needed because Spring's application context serves as the Citrus registry.

# Testing a SQL Polling Consumer — the full flow

The SQL Polling Consumer is a Camel route that polls a database table for new rows, publishes them to Kafka, and updates their status to prevent reprocessing. 
Here is the route:

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
            .to("kafka:eip.orders.placed?brokers={{kafka.brokers}}");
    }
}
```

The route does three things:

1. **SELECT** — queries for rows with status `PLACED`, ordered by creation time, limited to 10 per poll.
2. **Publish** — marshals each row to JSON and sends it to the `eip.orders.placed` Kafka topic.
3. **onConsume** — after successfully publishing, updates the row's status to `PROCESSING` so it won't be picked up again.

Testing this flow requires inserting a row, waiting for the route to poll it, verifying the Kafka message, and then checking that the database status changed. 

Here is the complete test:

```java
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
```

Let's walk through each step.

## Step 1: Insert test data

```java
t.when(
    sql(dataSource)
        .statement("INSERT INTO orders.orders "
            + "(customer_id, item_sku, quantity, amount) "
            + "VALUES ('CUST-00${id}', 'SKU-${id}', '1', '${amount}')")
);
```

The `sql(dataSource).statement(...)` action executes a SQL INSERT using the injected DataSource. Citrus variables (`${id}`, `${amount}`) are resolved before execution, producing unique test data for each run. The `id` column is not specified — PostgreSQL generates it with the `SERIAL` auto-increment.

## Step 2: Receive the Kafka message and capture the generated ID

```java
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
```

The `receive`action is wrapped in `repeatOnError()` because the SQL Polling Consumer runs on a timer — the route might not have polled the new row yet when the test first tries to read from Kafka. The retry loop gives the route up to 15 seconds to pick up and publish the row.

The body template contains three types of field references:

- **`${id}`** and **`${amount}`** — standard Citrus variables, resolved to the values set in `createVariables`. These validate that the route preserved the original data.
- **`@variable(order_id)@`** — captures the actual value of the `id` field from the received message and stores it in a new variable named `order_id`. This is the auto-generated database primary key that we need for the next step.
- **`@ignore@`** — accepts any value for the `created_at` field. Timestamps are non-deterministic, so we skip validation.

## Step 3: Verify the database state changed

```java
t.then(
    sql(dataSource)
        .query()
        .statement("SELECT status FROM orders.orders WHERE id = '${order_id}'")
        .validate("status", "PROCESSING")
);
```

The `sql(dataSource).query()` action executes a SELECT and validates the result. 
The `${order_id}` variable — captured from the Kafka message in step 2 — identifies the exact row to check. The `.validate("status", "PROCESSING")` assertion confirms that the route's `onConsume` callback updated the status from `PLACED` to `PROCESSING`.

This is the critical verification that many tests skip. The route might successfully publish to Kafka but fail the `onConsume` update — a bug that causes the same row to be published again on the next poll cycle. Only a database assertion catches this.

# Capturing auto-generated values with @variable()@

The `@variable(name)@` notation is one of Citrus's most useful features for database testing. 
When it appears in a received message body, Citrus extracts the actual value at that position and stores it in a test variable. 
After extraction, the variable is available for all subsequent actions.

Compare `@variable()@` with `@ignore@`:

| Marker             | Behavior                                               | Use when                                                                   |
|--------------------|--------------------------------------------------------|----------------------------------------------------------------------------|
| `@variable(name)@` | Accepts any value, **captures** it as a named variable | You need the value later (database IDs, transaction IDs, correlation keys) |
| `@ignore@`         | Accepts any value, **discards** it                     | You don't need the value (timestamps, UUIDs you won't reference again)     |

In the SQL Polling Consumer test, `@variable(order_id)@` captures the auto-generated primary key so it can be used in the subsequent `SELECT` query. 
Without this mechanism, you would need to query the database first to find the generated ID, adding complexity and a race condition.

# Database testing in YAML DSL

One of the most significant features in Citrus is that it supports multiple test domain specific languages.
In the sections before we have been using Java DSL that can be combined with arbitrary JUnit Jupiter tests on top of Quarkus or Spring Boot.

The same SQL Polling Consumer test can be written in pure YAML with a zero code approach (e.g. for Camel CLI projects):

```yaml
# deps: org.postgresql:postgresql:42.7.5
name: sql-polling-consumer-test
description: Test SQL Polling Consumer — polls PostgreSQL and publishes to Kafka
variables:
  - name: kafka.broker
    value: localhost:9092
  - name: id
    value: "citrus:randomNumber(4)"
  - name: amount
    value: 40
configuration:
  beans:
    - name: dataSource
      type: org.postgresql.ds.PGSimpleDataSource
      properties:
        url: "jdbc:postgresql://localhost:5432/eip"
        user: "eip"
        password: "eip"
actions:
  - testcontainers:
      compose:
        up:
          file: "_infra/compose.yaml"
  - waitFor:
      timeout: "25000"
      http:
        url: http://localhost:8090
  - camel:
      jbang:
        run:
          integration:
            name: "sql-polling-consumer"
            file: "../sql-polling-consumer.yaml"
            systemProperties:
              file: "../application.properties"
  - camel:
      jbang:
        verify:
          integration: "sql-polling-consumer"
          logMessage: "Routes startup"

  # Insert test data
  - sql:
      dataSource: "dataSource"
      statements:
        - statement: >-
            INSERT INTO orders.orders (customer_id, item_sku, quantity, amount)
            VALUES ('CUST-00${id}', 'SKU-${id}', 1, ${amount})

  # Receive from Kafka and capture the generated ID
  - repeatOnError:
      until: i >= 15
      autoSleep: 1000
      actions:
      - receive:
          endpoint: >-
            kafka:eip.orders.placed?server=${kafka.broker}&consumerGroup=citrus-placed-group
          timeout: 60000
          message:
            body:
              data: |
                {
                  "id": "@variable(order_id)@",
                  "customer_id": "CUST-00${id}",
                  "status": "PLACED",
                  "amount": ${amount}.0,
                  "item_sku": "SKU-${id}",
                  "quantity": 1,
                  "created_at": "@ignore@"
                }

  # Verify the database status changed
  - sql:
      dataSource: "dataSource"
      statements:
        - statement: "SELECT status FROM orders WHERE id = '${order_id}'"
      validate:
        - column: "status"
          value: "PROCESSING"
```

A few YAML-specific details:

- The `# deps:` comment at the top declares the PostgreSQL JDBC driver as a Maven dependency. Camel JBang resolves it automatically — no `pom.xml` needed.
- The `configuration.beans` section declares a `PGSimpleDataSource` bean. In Java tests, the DataSource comes from the application's CDI or Spring context. In YAML tests, you declare it explicitly.
- SQL validation uses `column`/`value` pairs in the `validate` block, compared to Java's `.validate("column", "value")` chained calls.

# Transactional Outbox Pattern

The SQL Polling Consumer test follows a DB → Kafka → DB flow. 
The reverse is equally common: Kafka → route → DB. When a route consumes from Kafka, processes the message, and writes to the database, the test needs to verify that the database write happened. 
But the write operation is asynchronous — the message might still be in-flight when the test queries the database.

The transactional outbox pattern demonstrates this scenario. 
The route consumes a payment request from Kafka, writes a payment record and an outbox event in a single transaction, and a second route polls the outbox and publishes to Kafka:

```java
from("kafka:eip.payments.required?brokers={{kafka.brokers}}&groupId=payment-outbox")
    .routeId("transactional-client")
    .unmarshal().json()
    .transacted()
    .to("sql:INSERT INTO payments.payments (order_id, amount, status) "
        + "VALUES (:#${body[order_id]}, :#${body[amount]}, 'PROCESSED')")
    // ... build outbox event ...
    .to("sql:INSERT INTO payments.outbox (event_id, event_type, aggregate_id, payload) "
        + "VALUES (:#${body[event_id]}, :#${body[event_type]}, "
        + ":#${body[aggregate_id]}, :#${body[payload]})");

from("sql:SELECT * FROM payments.outbox WHERE published = false "
        + "ORDER BY created_at LIMIT 100"
        + "?delay=5000"
        + "&onConsume=UPDATE payments.outbox SET published = true "
        + "WHERE event_id = :#event_id")
    .routeId("outbox-publisher")
    .setBody(simple("${body[payload]}"))
    .to("kafka:eip.payments.processed?brokers={{kafka.brokers}}");
```

The test sends a payment request to Kafka and then needs to verify three things: the payment row exists in the database, the payment event was published to Kafka, and the outbox row was marked as published. Each check wraps its assertion in `repeatOnError()`:

```java
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

        // Verify the payment row was written
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

        // Verify the outbox event was published to Kafka
        t.then(
            repeatOnError()
                .until((i, context) -> i > 20)
                .autoSleep(Duration.ofSeconds(2))
                .actions(
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
                )
        );

        // Verify the outbox row was marked as published
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
```

The pattern is the same each time: wrap the assertion in `repeatOnError()` with a retry loop. 
The first SQL query retries up to 15 times with 1-second intervals, giving the route 15 seconds to process the Kafka message and write to the database. 
The Kafka receive retries with 2-second intervals because the outbox publisher polls on a 5-second delay. 
The final SQL query verifies the outbox cleanup.

Without the retry wrapper, these assertions would fail intermittently — the test queries the database before the route finishes writing. 
This is the most common cause of flaky database tests, and `repeatOnError()` eliminates it.

# Best practices for database testing

After working through these examples, a few patterns emerge that keep database tests reliable:

**Use unique test data.** Generate random IDs with `citrus:randomNumber()` so each test run operates on its own rows. 
This prevents cross-test interference when tests run in sequence without a database reset.

**Always verify the database state change.** Don't just check the Kafka output — verify that the database update happened too. 
A route that publishes to Kafka but fails the `onConsume` UPDATE will reprocess the same row on the next poll cycle. 
Only a database assertion catches this.

**Wrap async queries in repeatOnError.** When a route writes to the database asynchronously (after consuming from Kafka), the INSERT may not have landed when the test queries. 
The `repeatOnError()` retry loop eliminates this race condition.

**Use @ignore@ for timestamps, @variable()@ for generated keys.** Timestamps are non-deterministic — skip them. 
Auto-generated IDs are non-deterministic but needed downstream — capture them.

**Define schemas in init scripts, not application auto-DDL.** The `init-schemas.sql` script in `docker-entrypoint-initdb.d/` runs before any test. 
This is more reliable than relying on the application's auto-DDL, which might create schemas in a different order or skip tables the test needs.

# Where to find the examples

The complete source code is available in the [EIP with Camel](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing) repository:

- **SQL Polling Consumer test**: [EipTests.java](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/14-consumer-patterns/quarkus/src/test/java/com/example/eip/consumers/EipTests.java) (`SqlPollingConsumerTest` nested class) and the [route definition](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/14-consumer-patterns/quarkus/src/main/java/com/example/eip/consumers/SqlPollingConsumerRoute.java).
- **Transactional Outbox test**: [EipTests.java](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/15-endpoints/quarkus/src/test/java/com/example/eip/endpoints/EipTests.java) (`TransactionalOutboxTest` nested class) and the [Outbox route](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/15-endpoints/quarkus/src/main/java/com/example/eip/endpoints/OutboxPatternRoute.java).
- **Database infrastructure**: [compose.yaml](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/14-consumer-patterns/quarkus/src/test/resources/_infra/compose.yaml) with PostgreSQL and the [init-schemas.sql](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/14-consumer-patterns/quarkus/src/test/resources/_infra/postgres/init-schemas.sql) script.
- **YAML DSL database test**: [sql-polling-consumer.citrus.it.yaml](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/14-consumer-patterns/yaml-dsl/test/sql-polling-consumer.citrus.it.yaml) with DataSource bean configuration.
- **EipTestSupport**: [EipTestSupport.java](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/14-consumer-patterns/quarkus/src/test/java/com/example/eip/consumers/EipTestSupport.java) with `assertProcessedExchanges` and `waitForCamelRouteStarted` helpers.

Give it a try, and let us know what you think!
