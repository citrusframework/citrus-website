---
layout: post
title: Camel CLI test plugin - Zero-Code Integration Tests
short-title: Camel CLI test plugin
author: Christoph Deppisch
github: christophd
categories: [blog]
---

[Apache Camel](https://camel.apache.org)'s YAML DSL lets you define integration routes without writing Java. [Camel JBang](https://camel.apache.org/manual/camel-jbang.html) lets you run them without a build tool. But how do you test them? Writing a Java test class with JUnit, `@QuarkusTest`, and Maven feels like overkill when the route itself is a single 20-line YAML file.

[Citrus](https://citrusframework.org) supports a YAML test DSL that mirrors this philosophy. You define your tests in `*.citrus.it.yaml` files alongside your YAML routes and run them with `camel test`. The test files follow the same declarative style as the routes — no Java, no Maven, no build configuration. Infrastructure starts with Docker Compose, messages flow through real Kafka topics, REST endpoints receive real HTTP requests, and database rows get inserted and verified — all in YAML.

This post walks through the full range of YAML DSL testing: Kafka message validation, REST endpoint testing, database interactions, and the infrastructure lifecycle that ties it all together.

# The YAML test file structure

A Citrus YAML test file has four parts: a name, a description, optional variables, and a list of actions that execute sequentially:

```yaml
name: OrderValidationTest
description: "Verify order validation route correctly routes valid and invalid orders"
actions:
  - send:
      endpoint: "kafka:eip.orders.incoming"
      message:
        body: |
          {"orderId":"ORD-001","customerId":"C-101","item":"Shipping Container","quantity":2}
  - receive:
      endpoint: "kafka:eip.orders.validated"
      timeout: 10000
      message:
        body: |
          {"orderId":"ORD-001","customerId":"C-101","item":"Shipping Container","quantity":2}
```

The file uses the naming convention `<name>.citrus.it.yaml` — the `.citrus.it.yaml` suffix tells the Camel test runner to treat it as a Citrus integration test. Actions run top to bottom: each action must complete before the next one starts. If any action fails, the test stops immediately with a validation error.

You run the test with the Camel CLI:

```bash
camel test run test/order-validation-test.yaml
```

No `pom.xml`, no `mvn verify`, no JUnit runner. The Camel test plugin handles dependency resolution, Citrus initialization, and test execution.

# An example - Testing Kafka routes

## The Camel route under test

Here is an order validation route written in YAML DSL. 
It consumes orders from Kafka, checks for required fields, and routes valid orders to `eip.orders.validated` and invalid ones to `eip.orders.rejected`:

```yaml
- route:
    id: order-validation
    from:
      uri: kafka:eip.orders.incoming
      steps:
        - unmarshal:
            json:
              unmarshalType: java.util.Map
        - choice:
            when:
              - simple: "${body[orderId]} == null"
                steps:
                  - setHeader:
                      name: rejectionReason
                      constant: "orderId must not be null"
                  - marshal:
                      json: {}
                  - to:
                      uri: kafka:eip.orders.rejected
              - simple: "${body[customerId]} == null"
                steps:
                  - setHeader:
                      name: rejectionReason
                      constant: "customerId must not be null"
                  - marshal:
                      json: {}
                  - to:
                      uri: kafka:eip.orders.rejected
              - simple: "${body[item]} == null"
                steps:
                  - setHeader:
                      name: rejectionReason
                      constant: "item must not be null"
                  - marshal:
                      json: {}
                  - to:
                      uri: kafka:eip.orders.rejected
              - simple: "${body[quantity]} == null || ${body[quantity]} <= 0"
                steps:
                  - setHeader:
                      name: rejectionReason
                      constant: "quantity must be greater than zero"
                  - marshal:
                      json: {}
                  - to:
                      uri: kafka:eip.orders.rejected
            otherwise:
              steps:
                - setHeader:
                    name: validatedAt
                    simple: "${date:now:yyyy-MM-dd'T'HH:mm:ss.SSSZ}"
                - marshal:
                    json: {}
                - to:
                    uri: kafka:eip.orders.validated
```

The route has five branches: four rejection cases (missing `orderId`, `customerId`, `item`, or zero/missing `quantity`) and one success path. A thorough test covers all of them.

With the Camel CLI tooling you can just run this single file with no further project setup.
This is a perfect scene for fast prototyping and experimenting.

```bash
camel run order-validation.yaml
```

## The Citrus YAML test

Citrus can use pure YAML test definitions in the same approach for fast prototyping and declarative testing without any further project setup.

The Citrus YAML test can use the full capabilities of the framework with test actions and validation steps as we know it from the Java DSL.

```yaml
name: OrderValidationTest
description: "Verify order validation route correctly routes valid and invalid orders"
actions:
  # --- Scenario 1: valid order ---
  - send:
      endpoint: "kafka:eip.orders.incoming"
      message:
        body: |
          {"orderId":"ORD-001","customerId":"C-101","item":"Shipping Container","quantity":2}
  - receive:
      endpoint: "kafka:eip.orders.validated"
      timeout: 10000
      message:
        body: |
          {"orderId":"ORD-001","customerId":"C-101","item":"Shipping Container","quantity":2}

  # --- Scenario 2: missing customerId ---
  - send:
      endpoint: "kafka:eip.orders.incoming"
      message:
        body: |
          {"orderId":"ORD-002","item":"Pallet Jack","quantity":1}
  - receive:
      endpoint: "kafka:eip.orders.rejected"
      timeout: 10000
      message:
        body: |
          {"orderId":"ORD-002","item":"Pallet Jack","quantity":1}

  # --- Scenario 3: zero quantity ---
  - send:
      endpoint: "kafka:eip.orders.incoming"
      message:
        body: |
          {"orderId":"ORD-003","customerId":"C-102","item":"Cargo Net","quantity":0}
  - receive:
      endpoint: "kafka:eip.orders.rejected"
      timeout: 10000
      message:
        body: |
          {"orderId":"ORD-003","customerId":"C-102","item":"Cargo Net","quantity":0}

  # --- Scenario 4: missing item ---
  - send:
      endpoint: "kafka:eip.orders.incoming"
      message:
        body: |
          {"orderId":"ORD-004","customerId":"C-103","quantity":5}
  - receive:
      endpoint: "kafka:eip.orders.rejected"
      timeout: 10000
      message:
        body: |
          {"orderId":"ORD-004","customerId":"C-103","quantity":5}
```

Each scenario follows the same pattern: `send` a message to the input topic, `receive` from the expected output topic. The `timeout: 10000` gives the route up to 10 seconds to process the message — if no message arrives on the expected topic within that window, the test fails.

The body validation is an exact JSON match. Citrus parses both the expected and actual JSON, compares every field, and reports any found mismatch. If the route changed a field value or dropped a field during processing, the validation catches it.

# Starting test infrastructure

The route under test connects to a Kafka message broker.
We need to prepare this test infrastructure locally before starting the Camel route and the test. Or even more comfortable we let the Citrus test care about the test infrastructure provisioning.

When the test needs real infrastructure — a Kafka broker, a database, a Pulsar cluster — you can start it with the `testcontainers` action and wait for its readiness:

```yaml
actions:
  - testcontainers:
      compose:
        up:
          file: "_infra/compose.yaml"
  - waitFor:
      timeout: "25000"
      http:
        url: http://localhost:8090
```

The `testcontainers.compose.up` action starts the Docker Compose stack defined in `_infra/compose.yaml`. The `waitFor.http` action polls the given URL — in this case kafka-ui on port 8090 — for up to 25 seconds. Once kafka-ui responds, we know Kafka is ready.

The compose file lives in the `_infra/` directory relative to the test file. This keeps infrastructure definitions separate from test logic and makes them reusable across multiple test files.

Here is the compose file that starts a Kafka broker in KRaft mode (no ZooKeeper) together with a kafka-ui instance for health checking:

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
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093,DOCKER://0.0.0.0:9094
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092,DOCKER://kafka:9094
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT,DOCKER:PLAINTEXT
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
      - KAFKA_AUTO_CREATE_TOPICS_ENABLE=true
      - CLUSTER_ID=RUlQQ2FtZWxQYXR0ZXJucw
    healthcheck:
      test: ["CMD-SHELL", "nc -z localhost 9092"]
      interval: 5s
      timeout: 5s
      retries: 12
      start_period: 30s

  kafka-ui:
    image: docker.io/provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    environment:
      - KAFKA_CLUSTERS_0_NAME=eip-local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9094
    depends_on:
      kafka:
        condition: service_healthy
```

The kafka-ui service depends on a healthy Kafka broker and exposes port 8090. That is why the `waitFor.http` action in the test polls `http://localhost:8090` — once kafka-ui responds, the entire Kafka stack is ready for the test to send and receive messages.

# Launching Camel route integrations

If you are running your tests as part of a full Quarkus or Spring Boot project you are used to leverage the fact that these frameworks automatically start the application under test with each test.

When prototyping with the YAML DSL the Citrus tests take care of launching the Camel route explicitly

You can use the Camel CLI test actions to run an integration as part of the test:

```yaml
  - camel:
      cli:
        run:
          integration:
            name: "order-validation"
            file: "../order-validation.yaml"
            systemProperties:
              file: "../application.properties"
```

The `camel.cli.run` action starts a Camel JBang integration as a subprocess managed by Citrus. The `name` identifies the integration for later verification. The `file` points to the YAML route definition, and `systemProperties.file` loads configuration (Kafka broker addresses, REST ports, etc.) from an application properties file.

This makes sure to start the Camel integration as part of the test.
It also configures the Camel integration with the local Kafka message broker and other connection properties from the previous Testcontainers infrastructure setup.

# Verifying Camel integration logs

An advantage of starting the Camel integration as part of the Citrus YAML test is that we can access the log output of the sub-process. 
This means you can verify that the route processed the message data correctly by checking its log output:

```yaml
  - camel:
      cli:
        verify:
          integration: "order-validation"
          logMessage: "Order validation successful"
```

The `camel.cli.verify` action scans the integration's log for the specified message. If the route logged `Order validation successful` during processing, the verification passes. This is a quick way to confirm the route received and processed a message without needing a separate `receive` action on the output topic.

To see the full integration output during test runs, add this setting to `citrus-application.properties` in your test directory:

```properties
citrus.camel.jbang.dump.integration.output=true
```

This is also a great way to inspect the Camel integration log output after the test for the sake of proper debugging and failure analysis.

# Testing REST endpoints

For routes that expose REST endpoints, Citrus provides HTTP client actions. Here is a test that validates a REST API for order processing:

```yaml
name: OrderHttpValidationTest
description: "Verify order validation via REST endpoint"
actions:
  # --- Scenario 1: valid order via POST ---
  - http:
      client: "orderApi"
      send:
        method: POST
        url: "http://localhost:8088/api/orders"
        message:
          headers:
            Content-Type: application/json
          body: |
            {"orderId":"ORD-010","customerId":"C-110","item":"Cargo Net","quantity":5}
  - http:
      client: "orderApi"
      receive:
        status: 200
        message:
          body: |
            {"status":"accepted","orderId":"ORD-010"}

  # --- Scenario 2: invalid order (missing item) ---
  - http:
      client: "orderApi"
      send:
        method: POST
        url: "http://localhost:8088/api/orders"
        message:
          headers:
            Content-Type: application/json
          body: |
            {"orderId":"ORD-011","customerId":"C-111","quantity":3}
  - http:
      client: "orderApi"
      receive:
        status: 400
        message:
          body: |
            {"status":"rejected","orderId":"ORD-011","reason":"item must not be null"}
```

The `http.client.send` action sends a POST request with a JSON body. The `http.client.receive` action waits for the response and validates both the HTTP status code and the response body. The `client: "orderApi"` parameter names a logical HTTP client — Citrus manages the connection and reuses it across send/receive pairs.

The test covers both the happy path (valid order, 200 response) and the error path (missing field, 400 response). The response body is validated as an exact JSON match, ensuring the API returns the expected structure and values.

The REST port is configured in the route's `application.properties`:

```properties
camel.rest.port=8088
```

# Database testing

YAML DSL tests can interact with databases directly. This is powerful for testing SQL Polling Consumer routes — routes that poll a database table for new rows and publish them to Kafka.

```yaml
# deps: org.postgresql:postgresql:42.7.5
name: sql-polling-consumer-test
description: Test SQL Polling Consumer — polls PostgreSQL rows and publishes to Kafka
variables:
  - name: kafka.broker
    value: localhost:9092
  - name: order.id
    value: "citrus:randomNumber(4)"
  - name: order.amount
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
      cli:
        run:
          integration:
            name: "sql-polling-consumer"
            file: "../sql-polling-consumer.yaml"
            systemProperties:
              file: "../application.properties"
  - camel:
      cli:
        verify:
          integration: "sql-polling-consumer"
          logMessage: "Routes startup"

  # Insert a test order into PostgreSQL
  - sql:
      dataSource: "dataSource"
      statements:
        - statement: >-
            INSERT INTO orders.orders (customer_id, item_sku, quantity, amount)
            VALUES ('CUST-00${order.id}', 'SKU-${order.id}', 1, ${order.amount})

  # Verify the route picks up the row and publishes to Kafka
  - receive:
      endpoint: >-
        kafka:eip.orders.placed?server=${kafka.broker}&consumerGroup=citrus-placed-group
      timeout: 60000
      message:
        body:
          data: |
            {
              "id": "@variable(order.dbId)@",
              "customer_id": "CUST-00${order.id}",
              "status": "PLACED",
              "amount": ${order.amount}.0,
              "item_sku": "SKU-${order.id}",
              "quantity": 1,
              "created_at": "@ignore@"
            }

  # Verify the route's onConsume updated the status to PROCESSING
  - sql:
      dataSource: "dataSource"
      statements:
        - statement: "SELECT status FROM orders.orders WHERE id = '${order.dbId}'"
      validate:
        - column: "status"
          value: "PROCESSING"
```

This test does several interesting things.

## Declaring dependencies and beans

The `# deps:` comment at the top declares a Maven dependency — the PostgreSQL JDBC driver. Camel JBang resolves this dependency automatically when running the test. No `pom.xml` needed.

The `configuration.beans` section declares a `PGSimpleDataSource` bean named `dataSource`. This bean is registered in the Citrus context and referenced by name in `sql` actions. The connection properties match the PostgreSQL service defined in the Docker Compose file.

## SQL insert and validation

The first `sql` action inserts a test order into the database. Variables like `${order.id}` are resolved at runtime, creating a unique row for each test run.

After the route polls the row and publishes it to Kafka, the `receive` action validates the Kafka message. Two special markers appear in the body:

- **`@variable(order.dbId)@`** — extracts the `id` field value from the actual message and stores it in a variable named `order.dbId`. This captures the database-generated primary key so it can be used in subsequent assertions.
- **`@ignore@`** — accepts any value for the `created_at` field. Timestamps are non-deterministic, so we skip their validation.

The final `sql` action queries the database to verify that the route's `onConsume` callback updated the order status from `PLACED` to `PROCESSING`. The `validate` block compares the `status` column against the expected value.

This test covers the full pipeline: database → Camel route → Kafka topic → database update. All in YAML.

# Negative testing with expectTimeout

Proving that a message was *not* routed somewhere works the same way in YAML as in Java. Here is a Message Filter test that verifies low-value orders are dropped:

```yaml
  # --- Send a low-value order (amount=50) → should be filtered out ---
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
```

The `expectTimeout` action tries to consume from `eip.orders.high-value` for 5 seconds. If nothing arrives, the test passes — proving the filter correctly dropped the low-value order. If a message does arrive, the test fails.

This is the YAML equivalent of Java's `expectTimeout().endpoint(...).timeout(5000)`. The keyword mapping is straightforward: `wait` in YAML corresponds to `timeout` in Java.

# When to use YAML tests

YAML DSL tests are a natural fit for:

- **Camel CLI projects** — the route is YAML, the test is YAML, no build tool required.
- **Quick prototyping** — validate a route idea in minutes without setting up a Maven project.
- **Teams without Java expertise** — operations or DevOps teams maintaining integration routes.
- **Simple send/receive/validate flows** — the declarative syntax handles the most common test patterns cleanly.

Java tests are the better choice when you need:

- **Complex assertion logic** — custom predicates, programmatic validation, computed expected values.
- **Shared test utilities** — interfaces like `EipTestSupport` with reusable helper methods.
- **Camel context access** — ControlBus commands, access to MBean route statistics, Camel mock endpoints.

Both approaches use the same Citrus engine underneath. The YAML DSL is a frontend that maps to the same action classes as the Java API. You can start with YAML tests for a JBang prototype and migrate to Java tests when the project grows into a Quarkus or Spring Boot application — the testing patterns transfer directly.

# Designing tests visually with Kaoto

Writing YAML by hand works well for simple tests, but as test files grow — multiple scenarios, infrastructure setup, database interactions, nested validation — a graphical view can make the structure easier to understand and edit.

![Kaoto Logo](/img/assets/kaoto-citrus-integration/kaoto-logo.png){:width="600px"}

[Kaoto](https://kaoto.io) is a visual designer for Apache Camel integrations. 
It renders YAML DSL definitions as interactive flow diagrams where you can add, configure, and reorder steps by clicking instead of typing. 
Since Kaoto has added the Citrus YAML DSL schemas to its catalog the visual designer is able to handle Citrus YAML test files, too. 
Kaoto is able to visualize and edit the declarative structure of a Citrus test with full support f all test actions, action containers, functions and validation matcher.

![Citrus in Kaoto](/img/assets/kaoto-citrus-integration/test-icons.png){:width="700px" .center-image}

In Kaoto, a Citrus YAML test renders as a sequence of actions — `send`, `receive`, `http`, `sql`, `testcontainers` — each represented as a configurable node. You can:

- **Add test actions** from a palette — drag a `send` or `receive` action into the flow and configure endpoint URIs, message bodies, and headers through form fields instead of editing raw YAML.
- **Reorder actions** by moving action nodes up or down in the flow.
- **Configure validation** — set expected HTTP status codes, JSON response bodies, SQL column assertions, and timeout values through the action's property panel.
- **See the full test structure at a glance** — the flow diagram shows the complete test lifecycle from infrastructure startup through message exchange to final assertions.

This is especially useful for teams that are new to Citrus or prefer visual tooling. 
The graphical editor produces the same `*.citrus.it.yaml` files shown throughout this post — you can switch between the visual editor and the YAML source at any time. 
There is no lock-in: the file on disk is always standard Citrus YAML DSL.

Kaoto runs as an [online editor in your browser](https://kaotoio.github.io/kaoto/#/) or as a [VS Code extension](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-kaoto) — install it, open a `.citrus.it.yaml` file, and the visual editor loads automatically.

# Where to find the examples

The complete source code for all tests shown in this post is available in the [EIP with Camel](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main) repository:

- **Order validation example**: [order-validation-test.yaml](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/41-citrus-testing/test/order-validation-test.yaml), [order-http-test.yaml](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/41-citrus-testing/test/order-http-test.yaml), and the [route under test](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/41-citrus-testing/order-validation-route.yaml).
- **SQL Polling Consumer YAML test**: [Consumer Patterns example](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/14-consumer-patterns/yaml-dsl/test/sql-polling-consumer.citrus.it.yaml) with PostgreSQL database interactions.
- **Message Filter negative test**: [Routing Fundamentals example](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/yaml-dsl/test/message-filter.citrus.it.yaml) with `expectTimeout` in YAML.

Install the Camel test plugin with `camel plugin add test`, then run any test with `camel test run <file>`.

Give it a try, and let us know what you think!
