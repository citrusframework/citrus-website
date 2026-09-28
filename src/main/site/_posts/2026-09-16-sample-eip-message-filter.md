---
layout: sample
title: Testing the Message Filter Pattern with Citrus
name: message-filter
image: /img/icons/camel.png
folder: examples/09-routing-fundamentals
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

For the test infrastructure setup, shared test utilities, runtime wiring, dependencies, and how to run the tests, see the [Camel EIP examples](/samples/camel-eip/) overview page.

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
