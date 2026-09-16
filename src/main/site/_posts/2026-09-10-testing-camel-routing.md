---
layout: post
title: Testing Camel Routing - Content-Based Routers, Filters, and Splitters
short-title: Testing Camel Routing
author: Christoph Deppisch
github: christophd
categories: [blog]
---

Routing is the backbone of every Apache Camel application. 
A `choice()` inspects a field and sends the message down one of multiple branches. 
A `filter()` drops everything that does not match a given expression. 
A `split()` breaks a batch into individual messages. 
These patterns are simple to write in Apache Camel yet the routing logic requires proper testing coverage to ensure that the route is doing what it is supposed to do.

Routing bugs are silent. A misconfigured filter predicate passes everything through. 
A content-based router checks the wrong field and sends incoming orders to the wrong queue. 
A splitter produces empty messages because the JsonPath expression doesn't match the actual payload structure. 
Without thorough testing, these defects ship to production.

![Featured](/img/assets/testing-camel-routing/featured.png){:width="700px" .center-image}
*AI generated with Google Gemini*

The challenge is not just proving that the right messages arrive at the right destination. 
It is also proving that the *wrong* messages do *not* arrive. 
A filter test that only checks the happy path — high-value order passes through — tells you nothing about whether low-value orders are actually being skipped. 
[Citrus](https://citrusframework.org) as a test framework provides the tools to test both sides: `receive()` for verifying that a message has been routed to a destination (e.g. a Kafka topic), and `expectTimeout()` for verifying that it has been filtered successfully.

This post walks through three fundamental routing patterns — Content-Based Router, Message Filter, and Splitter — and shows how to write thorough Citrus integration tests for each. 
We test every branch, prove what gets filtered, and verify that a batch split produces the expected number of output messages.

# Testing a Content-Based Router

A Content-Based Router enterprise integration pattern examines the message content and sends it to one of several destinations based on a condition. 
Here is a Camel route that classifies orders as hazmat (hazardous material), international, or domestic for a specific country:

```java
@ApplicationScoped
public class ContentBasedRouterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={{kafka.brokers}}&groupId=routing-demo")
            .routeId("content-based-router")
            .unmarshal().json()
            .log("Order received: ${body[order_id]} to ${body[destination_country]}")
            .choice()
                .when(simple("${body[contains_hazmat]} == true"))
                    .log("HAZMAT order ${body[order_id]} → hazmat handler")
                    .marshal().json()
                    .to("kafka:eip.orders.hazmat?brokers={{kafka.brokers}}")
                .when(simple("${body[destination_country]} != 'US'"))
                    .log("International order ${body[order_id]} → customs")
                    .marshal().json()
                    .to("kafka:eip.orders.international?brokers={{kafka.brokers}}")
                .otherwise()
                    .log("Domestic order ${body[order_id]} → standard")
                    .marshal().json()
                    .to("kafka:eip.orders.domestic?brokers={{kafka.brokers}}")
            .end();
    }
}
```

The route has three branches: hazmat (hazardous material) orders go to `eip.orders.hazmat`, international orders go to `eip.orders.international`, and everything else goes to `eip.orders.domestic`. 
The evaluation order matters — a hazmat order from the UK goes to the hazmat topic, not the international one.

## Minimum one test per branch

A single happy-path test gives us 33% branch coverage. We need three tests, one for each routing decision:

```java
@Nested
class ContentBasedRouterTest {

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
            receive()
                .endpoint("kafka:eip.orders.domestic?consumerGroup=citrus-domestic-group")
                .message()
                .body(Resources.create("templates/order.json"))
        );
    }
}
```

This test sends a domestic order (country `US`, no hazmat) and verifies it arrives on `eip.orders.domestic`. 
The pattern is straightforward: set variables that define the routing outcome according to the business logic, send the data to the input topic, receive the event from the expected output topic.

_NOTE:_ You may wonder what the step `waitForCamelRouteStarted()` is all about. This is a helper method that waits for the Camel route with the id `content-based-router` to report the status `Started`. 
Citrus is able to leverage the exposed route statistics exposed by Camel management module. 
This way the test makes sure to start sending Kafka events only when the Camel route is ready to consume events. 
This prevents unstable tests where consumer startup timing conditions play a role.

Here is the code for the helper method `waitForCamelRouteStarted()`:

```java
default TestActionBuilder<?> waitForCamelRouteStarted(String routeId, CamelContext camelContext) {
    return repeatOnError()
            .until((i, context) -> i > 20)
            .autoSleep(Duration.ofSeconds(1))
            .actions(
                camel().camelContext(camelContext)
                        .controlBus()
                        .route(routeId)
                        .status()
                        .result(ServiceStatus.Started)
                        .description("Waiting for Camel route '%s' to be started ...".formatted(routeId)),
                sleep().seconds(5)
            );
}
```

The international and hazmat tests follow the same structure — only the variable values and the output topic change:

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
        receive()
            .endpoint("kafka:eip.orders.international?consumerGroup=citrus-international-group")
            .message()
            .body(Resources.create("templates/order.json"))
    );
}

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
        receive()
            .endpoint("kafka:eip.orders.hazmat?consumerGroup=citrus-hazmat-group")
            .message()
            .body(Resources.create("templates/order.json"))
    );
}
```

All three tests share the same input topic (`eip.orders.placed`) and the same message template. 
The only differences are the variable values that control which branch the router takes and the output topic where we expect the message to appear.
With JUnit Jupiter we could also think about using a parameterized test that just changes the values for `country` and `hazmat` with the expected output topic name as another parameter.

## Template-based validation

The message template at `src/test/resources/templates/order.json` is the same for all three test variations. 
The template uses Citrus variable placeholders so each variation can place its individual values into the template:

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

The same template also serves both sending and receiving. 
When Citrus sends an event, it resolves the placeholders to produce the actual JSON. 
When Citrus receives an event, it resolves the same placeholders and compares every field against the actual message from Kafka. 
If the route changed any value — for example, if a buggy transformation overwrote the `amount` — the validation catches it immediately.

The `citrus:randomNumber(4)` function generates a unique 4-digit ID for each test run, preventing interference between test executions.
Citrus provides many functions to generate dynamic test data for better test stability where each test uses unique test data.
Each test operates on its own order, even if tests run in parallel or in quick succession.

## Kafka consumer group isolation

Each `receive()` action uses a unique `consumerGroup` parameter: `citrus-domestic-group`, `citrus-international-group`, `citrus-hazmat-group`. This is important for two reasons.

First, it prevents the test consumer from competing with the application's own consumer groups. 
The Camel route consumes from `eip.orders.placed` with group `routing-demo` — the test consumers read from the output topics with their own groups, so they never steal messages from each other.

Second, it prevents cross-test interference. If two tests read from the same topic with the same consumer group, Kafka's partition assignment can cause one test to miss messages that the other consumed. Unique consumer groups per test guarantee that each test sees all messages on its topic.

# Testing a Message Filter

A message filter enterprise integration pattern passes messages that match a condition and silently drops the rest. Here is a route that filters for high-value orders:

```java
@ApplicationScoped
public class MessageFilterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={{kafka.brokers}}&groupId=filter-demo")
            .routeId("message-filter")
            .unmarshal().json()
            .filter(simple("${body[amount]} >= 100"))
                .log("High-value order ${body[order_id]}: $${body[amount]}")
                .marshal().json()
                .to("kafka:eip.orders.high-value?brokers={{kafka.brokers}}")
            .end();
    }
}
```

Orders with `amount >= 100` pass through to `eip.orders.high-value`. Orders below that threshold are dropped — no output, no error, nothing.

## The positive test

The positive test is straightforward — send a high-value order and verify it arrives:

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

This proves the filter lets high-value orders through. But if we stop here, we have no evidence that the filter actually blocks anything. 
A broken predicate that passes *all* orders would still make this test green.

## The negative test with expectTimeout()

The negative test proves that a low-value order does *not* appear on the high-value topic:

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

The **`expectTimeout()`** action is the key here. It attempts to consume a message from the specified Kafka endpoint and waits for the given timeout (5 seconds). 
If *no* message arrives within that window, the test **passes**. 
If a message does arrive, the test **fails** — because it proves the filter let something through that it should have dropped.

This is the opposite of `receive()`. Where `receive()` asserts presence — a message must arrive — `expectTimeout()` asserts absence — no message must arrive.

_NOTE:_ A few things to watch for when using `expectTimeout()`:

- **Use a dedicated consumer group.** The `consumerGroup` is `citrus-filter-reject-group`, different from the positive test's `citrus-high-value-group`. If both tests used the same group, the positive test might consume the message before the negative test checks for it, causing a false pass.
- **The timeout must be long enough.** Five seconds is typically sufficient for a single-hop Kafka route. If the route does complex processing or calls external services, increase the timeout.
- **Watch for message ordering.** If `expectTimeout()` runs *after* a test that wrote to the same topic, it might pick up leftover messages from the earlier test. You may consider using very specific message selectors on the Kafka headers to selectively pick the right message form the topic. More on this in the closing section.

Proving something did *not* happen is fundamentally harder than proving something did. `expectTimeout()` makes this possible with a clear, declarative API — no manual thread sleeps or polling loops needed.

# Testing a Splitter

A splitter takes a single message containing multiple items and produces one output message per item. Here is a route that splits batch orders:

```java
@ApplicationScoped
public class SplitterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.batch?brokers={{kafka.brokers}}&groupId=splitter-demo")
            .routeId("order-splitter")
            .unmarshal().json()
            .log("Batch received with ${body[items].size()} items")
            .split(jsonpath("$.items[*]"))
                .log("Processing item: ${body[item_sku]} qty=${body[quantity]}")
                .marshal().json()
                .to("kafka:eip.orders.individual?brokers={{kafka.brokers}}")
            .end();
    }
}
```

The route consumes from `eip.orders.batch`, extracts each item from the `items` array using JsonPath, and publishes each one individually to `eip.orders.individual`.

## Sending a batch and receiving multiple outputs

The batch order template (`"templates/batch-order.json"`) sends two items:

```json
{
  "batch_id": "BATCH-${id}",
  "items": [
    { "item_sku": "PART-A1", "quantity": 2 },
    { "item_sku": "PART-B2", "quantity": 1 }
  ]
}
```

The test sends this order batch with two items and expects individual messages on the output topic based on the splitter EIP:

```java
@Nested
class SplitterTest {

    @Test
    public void shouldSplitBatchIntoIndividualItems() {
        t.given(
            createVariables()
                .variable("id", "citrus:randomNumber(4)")
        );

        t.given(waitForCamelRouteStarted("order-splitter", camelContext));

        t.when(
            send()
                .endpoint("kafka:eip.orders.batch")
                .message()
                .body(Resources.create("templates/batch-order.json"))
                .header("kafka.KEY", "BATCH-${id}")
        );

        t.then(
            createVariables()
                .variable("item_sku", "PART-A1"),
            receive()
                .endpoint("kafka:eip.orders.individual?consumerGroup=citrus-individual-group")
                .message()
                .body(Resources.create("templates/item.json"))
        ).and(
            createVariables()
                .variable("item_sku", "PART-B2"),
            receive()
                .endpoint("kafka:eip.orders.individual?consumerGroup=citrus-individual-group")
                .message()
                .body(Resources.create("templates/item.json"))
        );
    }
}
```

Both receives use the **same consumer group** (`citrus-individual-group`). This is intentional — we want the second receive to pick up where the first left off. If they used different consumer groups, both would try to read from the beginning of the topic, and Kafka's partition assignment might give both the same message.

## Validating the split items

The item template `"templates/item.json"` uses variables and `@ignore@` markers for fields whose specific values we don't need to assert:

```json
{
  "item_sku": "${item_sku}",
  "quantity": "@ignore@"
}
```

The `@ignore@` marker tells Citrus to accept any value for that field. 
This is useful when the split produces items with different field values — `PART-A1` with quantity 2 and `PART-B2` with quantity 1 — but we want a single template that validates both. 
The template asserts the message *structure* (has `item_sku` and `quantity` fields) without pinning the specific values.

_TIP:_ If you need to validate specific field values, you can use separate templates for each split item or use more test variables in the template. 
For many splitter tests, though, structural validation is sufficient — the important thing is that two messages arrived, both with the expected fields.

# Routing branch coverage as a testing discipline

The principle behind all of these tests is simple: **one test per routing branch, not just the happy path.** 
A `choice()` with three branches needs at least three tests. 
A `filter()` needs a positive test and a negative test. 
A splitter needs a test that verifies the correct number of output messages.

This might seem obvious, but in practice it is common to write a single test that exercises the default branch and call it done. 
Especially when verifying that something has **not** happened feels cumbersome. 
The problem is that routing bugs in the other branches remain hidden until production traffic triggers them. 
A Content-Based Router with a typo in the hazmat predicate — `${body[contains_hazmat]} = true` instead of `== true` — will pass the domestic test perfectly. 
Only the hazmat test variation catches it.

The investment is small. Each additional branch test follows the same structure as the first — change the input variables, change the expected output topic, done.
With parameterized tests you can even reuse the testing logic for multiple input and output variants.
The reward is high — complete confidence that every routing path produces the correct output.

# Where to find the examples

The complete source code for all tests shown in this post is available in the [EIP with Camel](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main) repository. The [Routing Fundamentals example](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals) includes:

- **Quarkus tests:** [EipTests.java](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/quarkus/src/test/java/com/example/eip/routing/EipTests.java) with Content-Based Router, Message Filter, Recipient List, and Splitter tests.
- **Spring Boot tests:** [EipTests.java](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/spring-boot/src/test/java/com/example/eip/routing/EipTests.java) with Content-Based Router, Message Filter, Recipient List, and Splitter tests.
- **Route definitions:** [ContentBasedRouterRoute.java](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/quarkus/src/main/java/com/example/eip/routing/ContentBasedRouterRoute.java), [MessageFilterRoute.java](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/quarkus/src/main/java/com/example/eip/routing/MessageFilterRoute.java), [SplitterRoute.java](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/quarkus/src/main/java/com/example/eip/routing/SplitterRoute.java).
- **YAML DSL tests:** [content-based-router.citrus.it.yaml](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/yaml-dsl/test/content-based-router.citrus.it.yaml), [message-filter.citrus.it.yaml](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/yaml-dsl/test/message-filter.citrus.it.yaml), [splitter.citrus.it.yaml](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/yaml-dsl/test/splitter.citrus.it.yaml).
- **Message templates:** [order.json](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/quarkus/src/test/resources/templates/order.json), [batch-order.json](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/quarkus/src/test/resources/templates/batch-order.json), [item.json](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/examples/09-routing-fundamentals/quarkus/src/test/resources/templates/item.json).

Clone the repository and run `mvn verify` in one of the runtime directories (e.g. `quarkus/`) to see all tests execute against real Kafka infrastructure.

Give it a try, and let us know what you think!
