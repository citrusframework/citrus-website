---
layout: sample
title: Testing the Content-Based Routers with Citrus
name: content-based-router
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Content-Based Router EIP in Apache Camel with Citrus across Quarkus, Spring Boot and YAML DSL
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/content-based-router/
---

Real-world messaging systems rarely route all messages to the same destination.
The application is supposed to handle various sets of messages differently based on defined business logic criteria.
The content-based router exactly provides this by sending messages to different destinations for further processing.

As an example, an order processing system needs to send hazardous materials to a specialized handler, international shipments to a customs pipeline, and domestic orders to the standard fulfillment center.
Each of these routing decisions depends on the content of the message itself — a field in the JSON body, a header value, or a combination of both.

This is the [Content-Based Router](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ContentBasedRouter.html) pattern, arguably the most-used routing pattern described in *Enterprise Integration Patterns* by Hohpe and Woolf.
It inspects each message and directs it to the appropriate channel based on what the message contains — essentially a message-level `if-else` or `switch` statement.

[Apache Camel](https://camel.apache.org) implements this pattern with the `choice()` EIP, which evaluates predicates top-to-bottom and routes each message to the first matching branch.
The implementation is straightforward, but testing it thoroughly is not.
A Content-Based Router with three branches requires at least three test scenarios — one for each branch — and each scenario must prove that the message arrives on the *correct* output channel and not on any of the others.
On top of that, predicate evaluation order matters: a hazmat order from the UK should go to the hazmat handler, not the international pipeline, even though it matches both predicates.

This is where [Citrus](https://citrusframework.org) comes in.
Citrus is an integration testing framework that speaks messaging natively — it can produce and consume Kafka messages, manage test infrastructure with Testcontainers, and validate that each routing decision sends the message to the right destination.
In this post, we walk through a complete example that tests a Content-Based Router with Citrus, running on Quarkus, Spring Boot, and the Camel YAML DSL.

# The Camel route under test

The route consumes orders from a Kafka topic `eip.orders.placed` and routes each order to one of three output topics based on its content:

- Orders that contain hazardous materials go to `eip.orders.hazmat`.
- International orders (destination country is not `US`) go to `eip.orders.international`.
- Everything else — domestic, non-hazmat orders — goes to `eip.orders.domestic`.

Here is the Camel route on Quarkus:

```java
@ApplicationScoped
public class ContentBasedRouterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=routing-demo")
            .routeId("content-based-router")
            .unmarshal().json()
            .log("Order received: ${body[order_id]} to ${body[destination_country]}")
            .choice()
                .when(simple("${body[contains_hazmat]} == true"))
                    .log("HAZMAT order ${body[order_id]} → hazmat handler")
                    .marshal().json()
                    .to("kafka:eip.orders.hazmat?brokers={% raw %}{{kafka.brokers}}{% endraw %}}")
                .when(simple("${body[destination_country]} != 'US'"))
                    .log("International order ${body[order_id]} → customs")
                    .marshal().json()
                    .to("kafka:eip.orders.international?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
                .otherwise()
                    .log("Domestic order ${body[order_id]} → standard")
                    .marshal().json()
                    .to("kafka:eip.orders.domestic?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
            .end();
    }
}
```

There are three important things to notice about this route.

First, the `choice()` evaluates predicates top-to-bottom and takes the **first match**.
Hazmat is checked before international, which means a hazmat order shipped to London goes to the hazmat handler, not the international pipeline.
This ordering is deliberate — hazardous materials require specialized handling regardless of destination, so the hazmat predicate must take priority.

Second, the route includes an `otherwise()` clause.
Without it, messages that don't match any `when` predicate would be silently dropped — a dangerous behavior in production.
The `otherwise()` acts as a catch-all, ensuring every message that enters the router reaches some output channel.

Third, the route marshals back to JSON before sending to the output topic.
The incoming message is unmarshalled into a `Map` for predicate evaluation, but the downstream consumers expect JSON.
This marshal-evaluate-marshal pattern is common in Camel routes that perform content-based routing on JSON payloads.

On Spring Boot, the route is identical except for the annotation:

```java
@Component
public class ContentBasedRouterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=routing-demo")
            .routeId("content-based-router")
            .unmarshal().json()
            .log("Order received: ${body[order_id]} to ${body[destination_country]}")
            .choice()
                .when(simple("${body[contains_hazmat]} == true"))
                    .log("HAZMAT order ${body[order_id]} → hazmat handler")
                    .marshal().json()
                    .to("kafka:eip.orders.hazmat?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
                .when(simple("${body[destination_country]} != 'US'"))
                    .log("International order ${body[order_id]} → customs")
                    .marshal().json()
                    .to("kafka:eip.orders.international?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
                .otherwise()
                    .log("Domestic order ${body[order_id]} → standard")
                    .marshal().json()
                    .to("kafka:eip.orders.domestic?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
            .end();
    }
}
```

The same routing logic expressed in the Camel YAML DSL:

```yaml
- route:
    id: content-based-router
    from:
      uri: "kafka:eip.orders.placed"
      parameters:
        brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
        groupId: routing-demo
      steps:
        - unmarshal:
            json:
              library: Jackson
        - log: "Order received: ${body[order_id]} to ${body[destination_country]}"
        - choice:
            when:
              - simple: "${body[contains_hazmat]} == true"
                steps:
                  - log: "HAZMAT order ${body[order_id]} → hazmat handler"
                  - marshal:
                      json:
                        library: Jackson
                  - to:
                      uri: "kafka:eip.orders.hazmat"
                      parameters:
                        brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
              - simple: "${body[destination_country]} != 'US'"
                steps:
                  - log: "International order ${body[order_id]} → customs"
                  - marshal:
                      json:
                        library: Jackson
                  - to:
                      uri: "kafka:eip.orders.international"
                      parameters:
                        brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
            otherwise:
              steps:
                - log: "Domestic order ${body[order_id]} → standard"
                - marshal:
                    json:
                      library: Jackson
                - to:
                    uri: "kafka:eip.orders.domestic"
                    parameters:
                      brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
```

All three variants express the same branching logic: hazmat first, then international, then domestic as the default.

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

This single template drives all three routing branches.
By changing `${country}` and `${hazmat}` between tests, we control exactly which `choice()` branch the router takes.
A domestic order sets `country=US` and `hazmat=false`; an international order sets `country=GB` and `hazmat=false`; a hazmat order sets `hazmat=true`.
Same template, same structure, different routing outcome — the variables are the only thing that changes.

# Testing the domestic branch

The first test verifies that a standard US order — no hazmat, domestic destination — is routed to the `eip.orders.domestic` topic.
This exercises the `otherwise()` branch, which is the default path when no other predicate matches.

```java
@Test
public void shouldRouteDomesticOrder() {
    t.given(
        createVariables()
            .variable("id", "citrus:randomNumber(4)")
            .variable("amount", 75.00)
            .variable("country", "US")
            .variable("hazmat", false)
    );

    t.given(waitForCamelRouteStarted("content-based-router", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .body(Resources.create("templates/order.json"))
            .header("kafka.KEY", "${id}")
    );

    t.then(
        repeatOnError()
            .times(25)
            .autoSleep(Duration.ofMillis(500))
            .actions(
                receive()
                    .endpoint("kafka:eip.orders.domestic?consumerGroup=citrus-domestic-group")
                    .message()
                    .body(Resources.create("templates/order.json"))
            )
    );
}
```

The test follows Citrus's given-when-then structure.
The `given` phase creates test variables — `country=US` and `hazmat=false` ensure neither the hazmat nor the international predicate matches — and waits for the route to be ready.
The `when` phase sends the order to the input topic.
The `then` phase receives and validates the message on the `eip.orders.domestic` output topic.

The `repeatOnError` wrapper handles the asynchronous nature of the Kafka-to-Kafka pipeline.
There is a small but unpredictable delay between producing a message on the input topic and seeing it on the output topic.
Rather than relying on a fixed sleep, `repeatOnError` polls: it attempts the receive up to 25 times with 500-millisecond pauses, succeeding as soon as the message arrives.

The receive action uses a dedicated consumer group (`citrus-domestic-group`) to avoid interfering with the application's own consumer groups.
It validates the full message body against the same template and variables used for sending — confirming that the message content passes through the router unchanged.

# Testing the international branch

The second test verifies that a non-US order is routed to the international topic.
We set `country=GB` and `hazmat=false` so the second `when` predicate — `${body[destination_country]} != 'US'` — is the first to match.

```java
@Test
public void shouldRouteInternationalOrder() {
    t.given(
        createVariables()
            .variable("id", "citrus:randomNumber(4)")
            .variable("amount", 80.00)
            .variable("country", "GB")
            .variable("hazmat", false)
    );

    t.given(waitForCamelRouteStarted("content-based-router", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .body(Resources.create("templates/order.json"))
            .header("kafka.KEY", "${id}")
    );

    t.then(
        repeatOnError()
            .times(25)
            .autoSleep(Duration.ofMillis(500))
            .actions(
                receive()
                    .endpoint("kafka:eip.orders.international?consumerGroup=citrus-international-group")
                    .message()
                    .body(Resources.create("templates/order.json"))
            )
    );
}
```

The structure is identical to the domestic test — only the input variables and the output topic differ.
This consistency is not accidental.
When every branch test follows the same shape — set variables, send to input, receive from expected output — the test suite becomes a table of routing rules that is easy to read, maintain, and extend when new branches are added.

# Testing the hazmat branch

The third test is the most interesting from a routing perspective.
We set `hazmat=true` and `country=US`, which means both the hazmat predicate and the domestic catch-all could plausibly apply.
Because `choice()` evaluates top-to-bottom and the hazmat predicate is listed first, the order should go to `eip.orders.hazmat` — not `eip.orders.domestic`.

```java
@Test
public void shouldRouteHazmatOrder() {
    t.given(
        createVariables()
            .variable("id", "citrus:randomNumber(4)")
            .variable("amount", 90.00)
            .variable("country", "US")
            .variable("hazmat", true)
    );

    t.given(waitForCamelRouteStarted("content-based-router", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .body(Resources.create("templates/order.json"))
            .header("kafka.KEY", "${id}")
    );

    t.then(
        repeatOnError()
            .times(25)
            .autoSleep(Duration.ofMillis(500))
            .actions(
                receive()
                    .endpoint("kafka:eip.orders.hazmat?consumerGroup=citrus-hazmat-group")
                    .message()
                    .body(Resources.create("templates/order.json"))
            )
    );
}
```

This test validates predicate evaluation order — a critical behavioral property of the Content-Based Router.
If someone reorders the `when` clauses in the route definition (putting international before hazmat, for example), this test would fail because a US hazmat order would fall through to the domestic branch instead of hitting the hazmat handler.
The test data is carefully crafted to expose exactly this kind of ordering bug: by combining `hazmat=true` with `country=US`, we ensure the message matches multiple predicates, and only the correct evaluation order produces the expected result.

# Testing with the YAML DSL

The Camel CLI supports running routes from YAML files and testing them with Citrus YAML test definitions.
The Content-Based Router test in YAML DSL covers all three branches in sequence:

```yaml
name: content-based-router-test
description: >-
  Test verifying the Content-Based Router pattern — routes orders
  to domestic, international, or hazmat topics
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
            name: "content-based-router"
            file: "../content-based-router.yaml"
            systemProperties:
              file: "../application.properties"

  # Domestic order
  - createVariables:
      variables:
        - name: id
          value: "citrus:randomNumber(4)"
        - name: amount
          value: "75.00"
        - name: country
          value: "US"
        - name: hazmat
          value: "false"
  - send:
      fork: true
      endpoint: >-
        kafka:eip.orders.placed?server=${kafka.broker}
      message:
        body:
          resource:
            file: "templates/order.json"
  - receive:
      endpoint: >-
        kafka:eip.orders.domestic?server=${kafka.broker}&consumerGroup=citrus-domestic-group
      message:
        body:
          resource:
            file: "templates/order.json"
  - camel:
      jbang:
        verify:
          integration: "content-based-router"
          logMessage: "Domestic order"

  # International order
  - createVariables:
      variables:
        - name: id
          value: "citrus:randomNumber(4)"
        - name: amount
          value: "80.00"
        - name: country
          value: "GB"
        - name: hazmat
          value: "false"
  - send:
      fork: true
      endpoint: >-
        kafka:eip.orders.placed?server=${kafka.broker}
      message:
        body:
          resource:
            file: "templates/order.json"
  - receive:
      endpoint: >-
        kafka:eip.orders.international?server=${kafka.broker}&consumerGroup=citrus-international-group
      message:
        body:
          resource:
            file: "templates/order.json"
  - camel:
      jbang:
        verify:
          integration: "content-based-router"
          logMessage: "International order"

  # Hazmat order
  - createVariables:
      variables:
        - name: id
          value: "citrus:randomNumber(4)"
        - name: amount
          value: "90.00"
        - name: country
          value: "US"
        - name: hazmat
          value: "true"
  - send:
      fork: true
      endpoint: >-
        kafka:eip.orders.placed?server=${kafka.broker}
      message:
        body:
          resource:
            file: "templates/order.json"
  - receive:
      endpoint: >-
        kafka:eip.orders.hazmat?server=${kafka.broker}&consumerGroup=citrus-hazmat-group
      message:
        body:
          resource:
            file: "templates/order.json"
  - camel:
      jbang:
        verify:
          integration: "content-based-router"
          logMessage: "HAZMAT order"
```

The YAML test is self-contained: it starts the test infrastructure, launches the Camel integration with `camel:jbang:run`, and runs all three routing scenarios in sequence.
Each scenario ends with a `camel:jbang:verify` action that checks the running integration's log output for the expected routing message — `"Domestic order"`, `"International order"`, or `"HAZMAT order"`.
This serves as a secondary confirmation that the correct `choice()` branch was taken, independent of which output topic received the message.

Notice the `fork: true` on each send action.
In the YAML DSL, `fork` tells Citrus to send the Kafka message and immediately proceed to the receive action without waiting for the producer acknowledgement to complete.
Without this, there is a timing risk: the test sends, waits for the Kafka ack, and then starts listening — but by that time the Camel route may have already processed the message and published it to the output topic.
The `fork` eliminates this race condition by overlapping the send and receive operations.

For the test infrastructure setup, shared test utilities, runtime wiring, dependencies, and how to run the tests, see the [Camel EIP examples](/samples/camel-eip/) overview page.

# Key takeaways

Testing a Content-Based Router means testing every branch — not just the happy path.
Each branch is a distinct routing decision, and the test suite should cover every one to prevent regressions when predicates are added, removed, or reordered.

Here is what Citrus brings to this problem:

- **One test per branch** — Each test sends a message crafted to trigger a specific `choice()` branch and verifies it arrives on the correct output topic. The test suite reads like a routing table: domestic, international, hazmat.
- **Variable-driven test data** — The same `order.json` template drives all three tests. By changing `${country}` and `${hazmat}`, you control which branch the router takes. Adding a new branch means adding a new test with different variable values — no new templates needed.
- **`repeatOnError`** — Polls for the expected message on the output topic without brittle fixed-duration sleeps. Succeeds as soon as the message arrives, keeping test execution time to a minimum.
- **Predicate order validation** — The hazmat test uses `hazmat=true` with `country=US` to ensure the hazmat predicate fires before the domestic catch-all. If someone reorders the `when` clauses, the test fails immediately.
- **Multi-runtime portability** — The same test DSL runs on Quarkus, Spring Boot, and the Camel CLI with YAML DSL. Only the wiring annotations change; the test logic stays identical.

The complete source code for this example is available on GitHub:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/spring-boot)
- [YAML DSL variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/yaml-dsl)

Give it a try, and let us know what you think!
