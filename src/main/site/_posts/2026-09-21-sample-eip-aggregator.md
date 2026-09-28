---
layout: sample
title: Testing the Aggregator Pattern with Citrus
name: aggregator
image: /img/icons/camel.png
folder: examples/13-aggregator
group: eip
description: Testing the Aggregator EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/aggregator/
---

In many of the patterns covered so far — content-based router, splitter, message filter — each incoming message is processed individually.
One message in, one routing decision out.
Many integration scenarios often work the other way around: *multiple* messages need to be combined into one before a downstream service can act on them.

Consider an order processing system where each line item is processed independently — inventory checked, price calculated, warehouse assigned.
Eventually, those individual results need to be reassembled into a single order response.
The fulfillment service cannot send 50 separate "your item shipped" emails — it needs one email listing all 50 items.

The [Aggregator](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Aggregator.html) pattern, described by Hohpe and Woolf, solves this.
It collects related messages by a *correlation key*, combines them using an *aggregation strategy*, and emits the result when a *completion condition* is met.
Three questions define every aggregator: which messages belong together, how are they combined, and when is the aggregation complete.

[Apache Camel](https://camel.apache.org) implements this with the `aggregate()` EIP — one of the most powerful and configurable patterns in the framework.
Camel's aggregator manages an internal repository of in-progress aggregations, correlates incoming messages by an expression, delegates combination logic to an `AggregationStrategy`, and flushes completed aggregations based on size, timeout, or a custom predicate.

Testing an aggregator is fundamentally different from testing stateless patterns.
The aggregator is *stateful*: it holds messages, tracks correlation groups, and only produces output when a completion condition fires.
A test must send multiple messages, wait for the aggregator to collect and combine them, and then verify the assembled result.
This time-dependent, multi-message behavior is exactly what makes integration testing with [Citrus](https://citrusframework.org) valuable — you exercise the full pipeline end-to-end against real infrastructure, proving that correlation, combination, and completion all work together.

## The scenario

Line items arrive on a Kafka topic `eip.orders.line-items` as individual JSON messages.
Each line item carries an `order_id`, a product SKU, a quantity, and a price.
The aggregator route collects line items that share the same `order_id`, assembles them into a complete order with a `line_items` array and a `total_amount`, and publishes the result to `eip.orders.complete`.

The aggregator uses `completionSize(3)` — it waits until three line items have been collected for the same order before flushing.
A `completionTimeout(10000)` acts as a safety net: if fewer than three items arrive within 10 seconds, the aggregator flushes whatever it has collected.

## The Camel route

### Quarkus

```java
@ApplicationScoped
public class AggregatorRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.line-items?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=aggregator-demo")
            .routeId("order-aggregator")
            .unmarshal().json()
            .log("Received line item for order ${body[order_id]}: ${body[item_sku]}")
            .aggregate(jsonpath("$.order_id"), new OrderAggregationStrategy())
                .completionSize(3)
                .completionTimeout(10000)
            .log("Aggregated complete order ${body[order_id]} with ${body[line_items].size} items, total=${body[total_amount]}")
            .marshal().json()
            .to("kafka:eip.orders.complete?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }

    private static class OrderAggregationStrategy implements AggregationStrategy {

        @Override
        public Exchange aggregate(Exchange oldExchange, Exchange newExchange) {
            var lineItem = newExchange.getIn().getBody(Map.class);

            if (oldExchange == null) {
                var order = new LinkedHashMap<String, Object>();
                order.put("order_id", lineItem.get("order_id"));
                order.put("customer_id", lineItem.get("customer_id"));
                order.put("line_items", new ArrayList<>(List.of(Map.of(
                    "item_sku", lineItem.get("item_sku"),
                    "quantity", lineItem.get("quantity"),
                    "price", lineItem.get("price")
                ))));
                order.put("total_amount", ((Number) lineItem.get("price")).doubleValue());
                order.put("status", "ASSEMBLED");
                newExchange.getIn().setBody(order);
                return newExchange;
            }

            var order = oldExchange.getIn().getBody(Map.class);
            var items = (List<Map<String, Object>>) order.get("line_items");
            items.add(Map.of(
                "item_sku", lineItem.get("item_sku"),
                "quantity", lineItem.get("quantity"),
                "price", lineItem.get("price")
            ));
            double total = ((Number) order.get("total_amount")).doubleValue()
                + ((Number) lineItem.get("price")).doubleValue();
            order.put("total_amount", total);
            return oldExchange;
        }
    }
}
```

The route reads from the `eip.orders.line-items` Kafka topic, unmarshals each message to a `Map`, and feeds it into the `aggregate()` EIP.
The correlation expression `jsonpath("$.order_id")` groups line items by their order ID.
The `OrderAggregationStrategy` handles the combination logic: on the first message for a given order, it initializes a new order structure with a `line_items` list and a running `total_amount`.
Each subsequent message for the same order adds its line item and increments the total.

### Spring Boot

The Spring Boot variant is identical in route logic — only the CDI annotation changes:

```java
@Component
public class AggregatorRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.line-items?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=aggregator-demo")
            .routeId("order-aggregator")
            .unmarshal().json()
            .log("Received line item for order ${body[order_id]}: ${body[item_sku]}")
            .aggregate(jsonpath("$.order_id"), new OrderAggregationStrategy())
                .completionSize(3)
                .completionTimeout(10000)
            .log("Aggregated complete order ${body[order_id]} with ${body[line_items].size} items, total=${body[total_amount]}")
            .marshal().json()
            .to("kafka:eip.orders.complete?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }

    // OrderAggregationStrategy is identical ...
}
```

`@Component` replaces `@ApplicationScoped` — the route logic stays the same across both runtimes.

## Completion conditions and why they matter for testing

Camel's aggregator supports multiple completion conditions that can be combined with OR semantics — whichever fires first triggers emission:

| Condition                   | Description                                       | Use case                     |
|-----------------------------|---------------------------------------------------|------------------------------|
| `completionSize(N)`         | Emit after N messages are collected               | Known number of parts        |
| `completionTimeout(ms)`     | Emit if no new message arrives within the timeout | Unknown number of parts      |
| `completionInterval(ms)`    | Emit on a fixed schedule                          | Periodic batching            |
| `completionPredicate(expr)` | Emit when a condition is true                     | All required stages complete |
| `forceCompletionOnStop()`   | Emit incomplete aggregations when route stops     | Graceful shutdown            |

For testing, these conditions directly shape the test design.
Our route uses `completionSize(3)` as the primary trigger and `completionTimeout(10000)` as a safety net.
If the test sends exactly three line items with the same order ID, the aggregator flushes immediately when the third arrives — no waiting.
If it sends fewer than three, the test must wait for the timeout.

This dual-condition behavior is exactly the kind of nuance that unit tests with mocked endpoints miss.
An integration test with real Kafka infrastructure surfaces the actual timing behavior.

## Message templates

The tests use JSON templates with Citrus variable placeholders for dynamic values.
Here is the line item template that represents individual messages sent to the aggregator:

```json
{
  "order_id": "ORD-${id}",
  "customer_id": "CUST-${id}",
  "item_sku": "${sku}",
  "quantity": ${quantity},
  "price": ${price}
}
```

And the expected output after aggregation — the complete order:

```json
{
  "order_id": "ORD-${id}",
  "customer_id": "CUST-${id}",
  "line_items": "@ignore@",
  "total_amount": "@ignore@",
  "status": "ASSEMBLED"
}
```

The `@ignore@` markers are a Citrus validation feature.
They tell the framework to accept any value for `line_items` and `total_amount` during message comparison.
The test validates the structural fields (`order_id`, `customer_id`, `status`) while deliberately ignoring the computed fields whose exact values depend on the combination of individual items.
This keeps the assertion focused: we care that the aggregator produced a complete order with the correct identity and status, not the exact JSON structure of the nested line items array.

## The aggregator test

The test sends three line items with the same order ID, then verifies that the aggregator produces a single complete order on the output topic.

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
    class AggregatorTest {

        @Test
        public void shouldAggregateThreeLineItemsIntoCompleteOrder() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-ABC-42")
                    .variable("quantity", 2)
                    .variable("price", 30)
            );

            t.given(waitForCamelRouteStarted("order-aggregator", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.line-items")
                    .message()
                    .body(Resources.create("templates/line-item.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "ORD-${id}")
            );

            t.when(
                createVariables()
                    .variable("sku", "SKU-DEF-77")
                    .variable("quantity", 1)
                    .variable("price", 50)
            );
            t.when(
                send()
                    .endpoint("kafka:eip.orders.line-items")
                    .message()
                    .body(Resources.create("templates/line-item.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "ORD-${id}")
            );

            t.when(
                createVariables()
                    .variable("sku", "SKU-GHI-13")
                    .variable("quantity", 3)
                    .variable("price", 20)
            );
            t.when(
                send()
                    .endpoint("kafka:eip.orders.line-items")
                    .message()
                    .body(Resources.create("templates/line-item.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "ORD-${id}")
            );

            t.then(
                repeatOnError()
                    .times(15)
                    .actions(
                        receive()
                            .endpoint("kafka:eip.orders.complete?consumerGroup=citrus-complete-group")
                            .message()
                            .body(Resources.create("templates/complete-order.json"))
                    )
            );
        }
    }
}
```

Let's walk through the test structure.

**Given — set up variables and wait for the route.**
The test generates a random 4-digit order ID that will be shared across all three line items, creating the correlation key.
The first set of variables defines the SKU, quantity, and price for the first line item.
It then waits for the `order-aggregator` route to reach `Started` status.

**When — send three correlated line items.**
Three line items go to the `eip.orders.line-items` Kafka topic.
Between each send, the test updates the `sku`, `quantity`, and `price` variables — but *not* the `id`.
This is how the test creates three distinct line items that share the same `order_id`: the template reads `${id}` for the order identity (which stays constant) and `${sku}`, `${quantity}`, `${price}` for the item-specific fields (which change).
All three messages use the same Kafka message key `ORD-${id}`, ensuring they land on the same partition for consistent ordering.

**Then — verify the aggregated output.**
The test receives the complete order from the `eip.orders.complete` topic and validates its body against the `complete-order.json` template.
The `repeatOnError()` block retries up to 15 times with 1-second intervals — giving the aggregator enough time to receive all three line items, combine them, and publish the result.

Because the test sends exactly three line items (matching the `completionSize(3)` threshold), the aggregator should flush almost immediately after the third message arrives.
The retry window is generous to account for Kafka consumer lag and Camel processing time, but in practice the assertion typically succeeds within 2-3 seconds.

For the test infrastructure setup, shared test utilities, runtime wiring, dependencies, and how to run the tests, see the [Camel EIP examples](/samples/camel-eip/) overview page.

## Persistent aggregation

The in-memory aggregation repository works fine for development and testing, but in production an application restart would lose all in-flight aggregations.
The example project includes a persistent aggregator variant that stores aggregation state in PostgreSQL via Camel's `JdbcAggregationRepository`.

```java
from("kafka:eip.orders.line-items?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=persistent-aggregator-demo")
    .routeId("persistent-order-aggregator")
    .unmarshal().json()
    .aggregate(jsonpath("$.order_id"), new PersistentOrderAggregationStrategy())
        .completionSize(3)
        .completionTimeout(15000)
        .aggregationRepository("#jdbcAggregationRepo")
    .marshal().json()
    .to("kafka:eip.orders.complete-persistent?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
```

The `aggregationRepository("#jdbcAggregationRepo")` directive tells Camel to store partial aggregation state in a PostgreSQL table rather than in memory.
If the service restarts mid-aggregation, the pending items are recovered from the database and aggregation resumes.

The test for the persistent aggregator follows the same structure — send three line items, receive one complete order — but validates against a template that includes a `persistent: true` marker:

```json
{
  "order_id": "ORD-${id}",
  "customer_id": "CUST-${id}",
  "line_items": "@ignore@",
  "total_amount": "@ignore@",
  "persistent": true
}
```

This simple marker field distinguishes persistent aggregator output from the in-memory variant, making it easy to verify that the correct route processed the messages.

## Why aggregator tests are different from stateless pattern tests

Aggregator tests require a fundamentally different approach compared to testing stateless patterns like a content-based router or a message filter:

**Multiple inputs produce a single output.**
Most pattern tests follow a 1:1 model — one message in, one message out (or one routing decision).
The aggregator flips this: N messages in, one message out.
The test must coordinate multiple sends and verify a single receive.

**The completion condition defines the test timing.**
With a stateless pattern, the output appears almost immediately after the input.
An aggregator holds messages until the completion condition fires.
When the test sends the exact number of messages matching `completionSize`, the output appears promptly.
When it sends fewer, the test must wait for the timeout — which is a valid test scenario but requires a longer retry window.

**Correlation is a first-class concern.**
The test must ensure that all input messages share the same correlation key.
If one line item has a different `order_id`, it starts a separate aggregation group and the expected completion never occurs.
The Citrus variable `${id}` — set once in the `given` phase and reused across all sends — naturally enforces this.

**Consumer group isolation prevents cross-test interference.**
The aggregator consumes from a shared topic, but each test uses a dedicated Citrus consumer group for receiving output.
This prevents one test from accidentally consuming another test's aggregated results.

## Key takeaways

- **Aggregators are stateful and multi-message.** Unlike stateless routing patterns, they accumulate messages before producing output. Integration tests must send multiple correlated messages and wait for the completion condition to fire.
- **Correlation keys are critical.** All messages that belong to the same aggregation group must share the same correlation expression value. In Citrus, setting the `${id}` variable once and reusing it across multiple sends naturally enforces this.
- **`completionSize` vs `completionTimeout` shapes the test.** Sending exactly N messages (matching the completion size) triggers immediate output. Sending fewer tests the timeout path — both are valid scenarios worth covering.
- **`@ignore@` keeps assertions focused.** Rather than asserting the exact structure of computed fields like `line_items` or `total_amount`, Citrus's ignore markers let you validate the structural identity of the aggregated message without brittle assertions on intermediate values.
- **`repeatOnError()` handles async verification gracefully.** Rather than inserting fixed sleeps, Citrus retries the receive action until it succeeds or the retry limit is reached — making tests both reliable and as fast as possible.
- **Persistent aggregation adds infrastructure but not test complexity.** The test structure for a JDBC-backed aggregator is identical to the in-memory variant. The difference is in the route configuration, not the test design.
- **Two runtimes, one test pattern.** Whether you run on Quarkus or Spring Boot, the test structure — send correlated items, wait for the aggregator, verify the assembled result — stays the same. Only the bootstrap annotations and infrastructure wiring change.
