---
layout: post
title: Testing Reliability Patterns - Dead Letter Queues, Idempotent Receivers, and Circuit Breakers
short-title: Testing Reliability Patterns
author: Christoph Deppisch
github: christophd
categories: [blog]
---

Reliability patterns exist for when things go wrong. 
Messages get duplicated, services become unavailable, processing throws exceptions. 
[Apache Camel](https://camel.apache.org) provides Dead Letter Channels, Idempotent Receivers, and Circuit Breakers to handle these situations. 
But there is an uncomfortable truth: if you only test the happy path, you have no idea whether your error handling actually works.

Testing failure scenarios is harder than testing success. 
How do you send a message that triggers a Dead Letter Channel? How do you verify that a duplicate was silently dropped? How do you assert on a route that logs an error but produces no output message? 
These questions matter because reliability patterns are invisible until something breaks — and production is the worst place to discover they are misconfigured.

[Citrus](https://citrusframework.org) provides the tools to test all of these scenarios: `repeatOnError` for polling DLQ topics, `assertProcessedExchanges` for verifying routes with no output topic, deterministic test data for triggering specific failure conditions, and ordered test methods for building stateful test sequences like deduplication.

![Citrus](/img/assets/testing-reliability-patterns/featured.png){:width="700px" .center-image}
*AI generated with Google Gemini*

# Testing Dead Letter Channels

A Dead Letter Channel catches messages that fail processing after exhausting all retry attempts and routes them to a separate destination — typically a DLQ topic — instead of losing them. 
The Camel route configuration is straightforward:

```java
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
            throw new RuntimeException("Simulated failure for order " + orderId);
        }
    })
    .log("Order ${body[order_id]} processed successfully")
    .marshal().json()
    .to("kafka:eip.orders.processed?brokers={{kafka.brokers}}");
```

This route processes orders from `eip.orders.placed`. Orders whose `order_id` is divisible by 5 throw an exception simulating errors. 
After 3 retries, the Dead Letter Channel routes the original message to `eip.orders.dlq`. 
The `useOriginalMessage()` option is important — it sends the message as it was *before* processing, not the partially-transformed version that failed.

Testing this pattern requires two tests: one that proves the happy path works, and one that deliberately triggers the failure path.

## Happy path: order processed successfully

The first test sends an order with `id=1001` — not divisible by 5, so processing succeeds:

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

_IMPORTANT:_ Notice the `fork(true)` on the send action. This forces Citrus to perform the Kafka send and the subsequent receive concurrently — Citrus sends the message and immediately starts listening for the event on the output topic. 
Without `fork(true)` we run into timing conditions as the test both produces an event and consumes the triggered output event. 
The test would send the message wait for the Kafka acknowledgement, then start listening, and potentially miss the processed message if the Camel route finishes before the Citrus consumer is ready.
This can lead to flaky tests where the timing condition may succeed or fail in an unpredictable way.

## Failure path: order routes to DLQ

The second test sends an order with `id=1000` — divisible by 5, triggering the exception:

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

The `repeatOnError` wrapper is essential here. The Dead Letter Channel retries the message 3 times with a 1-second delay before giving up. 
That means the DLQ message will not appear for at least 3 seconds. 
The test polls the DLQ topic up to 10 times, sleeping 1 second between attempts, until it finds the expected message.

There is a subtle point about what lands on the DLQ. Because the route uses `useOriginalMessage()`, the DLQ receives the message exactly as it was sent — the original JSON with the same `order_id`, `amount`, and other fields. 
Any fields the route would have added during successful processing (enrichment data, computed values) are absent. 
When writing assertions against DLQ messages, only assert on what was in the original input.

You can find the complete Dead Letter Channel route and test in the [05-reliability example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/05-reliability/quarkus).

# Testing Idempotent Receivers

An Idempotent Receiver prevents duplicate messages from being processed more than once. 
The pattern uses a unique identifier — an event ID, transaction ID, or message key — and checks it against a deduplication store before processing. 
If the key has been seen before, the message is either dropped or routed to a separate handler.

Here is an Idempotent Receiver route that uses Redis for deduplication:

```java
from("kafka:eip.orders.payments?brokers={{kafka.brokers}}&groupId=redis-idempotent")
    .routeId("idempotent-receiver")
    .unmarshal().json(Map.class)
    .process(this::deduplicateAndProcess)
    .choice()
        .when(header("CamelDuplicate").isEqualTo(true))
            .to("direct:handle-duplicate-payment")
        .otherwise()
            .log("Processing payment event ${body[event_id]} for order ${body[order_id]}")
            .marshal().json()
            .to("kafka:eip.orders.payment-confirmed?brokers={{kafka.brokers}}")
    .end();

from("direct:handle-duplicate-payment")
    .routeId("handle-duplicate-payment")
    .log("Duplicate payment event ${body[event_id]} -- skipping");
```

The `deduplicateAndProcess` method performs a Redis `SET NX` with a 24-hour TTL. 
If the key already exists (meaning this event was already processed), it sets the `CamelDuplicate` header to `true`. 
The route's `choice()` then branches: unique events go to `payment-confirmed`, duplicates go to `handle-duplicate-payment`.

Here is the deduplication logic:

```java
private void deduplicateAndProcess(Exchange exchange) {
    Map<String, Object> event = exchange.getIn().getBody(Map.class);
    String eventId = String.valueOf(event.get("event_id"));
    String dedupeKey = "idempotent:" + eventId;

    Response result = redis.set(List.of(dedupeKey, "1", "NX", "EX", "86400"))
        .await().indefinitely();

    boolean isDuplicate = (result == null);
    exchange.getIn().setHeader("CamelDuplicate", isDuplicate);
}
```

## Ordered tests for stateful deduplication

Testing deduplication requires something that most tests do not need: execution order. 
The first test must seed the deduplication key, and the second test must verify that the same key is recognized as a duplicate. 
JUnit 5's `@TestMethodOrder` and `@Order` annotations make this explicit:

```java
@Nested
class IdempotentReceiverTest {

    @Test
    public void shouldDropDuplicatePaymentEvent() {
        String paymentEvent = """
            {
              "event_id": "EVT-2001",
              "order_id": 2001,
              "amount": 149.99
            }
            """;

        t.given(waitForCamelRouteStarted("idempotent-receiver", camelContext));

        t.when(
            send()
                .endpoint("kafka:eip.orders.payments")
                .message()
                .body(paymentEvent)
                .header(KafkaMessageHeaders.MESSAGE_KEY, "PAY-2001")
        );

        t.then(
            receive()
                .endpoint("kafka:eip.orders.payment-confirmed?consumerGroup=citrus-payment-confirmed-group")
                .message()
                .body(paymentEvent)
        );

        t.when(
            send()
                .endpoint("kafka:eip.orders.payments")
                .message()
                .body(paymentEvent)
                .header(KafkaMessageHeaders.MESSAGE_KEY, "PAY-2001-DUP")
        );

        t.then(
            expectTimeout()
                .endpoint("kafka:eip.orders.payment-confirmed?consumerGroup=citrus-payment-confirmed-group")
                .timeout(5000)
        );

        t.then(
            camel().route()
                .verifyRouteStats("handle-duplicate-payment")
                .completed(1)
                .failed(0)
        );
    }
}
```

The first test step sends the payment event `EVT-2001` and verifies that it gets routed to the `payment-confirmed` topic. 
This seeds the key `idempotent:EVT-2001` in Redis. 
Then the test sends the exact same event again. 
This time, the Redis `SET NX` returns `null` (the key already exists), the route sets `CamelDuplicate=true`, and the message is routed to `handle-duplicate-payment` instead of `payment-confirmed`.
The `expectTimeout()` action verifies that the event has **not** been sent to the `payment-confirmed` topic.

In addition to that the test verifies that the `handle-duplicate-payment` route has been called. 
But here is a challenge: the `handle-duplicate-payment` route only logs — it produces no output message on any topic. 
There is nothing to `receive()`. This is where `verifyRouteStats` comes in.
It uses the Camel management route statistics to verify the number of completed and failed exchanges on that route.

_IMPORTANT:_ The `ManagedCamelContext` API that provides access to route statistics requires a management dependency. 
Without it, `getManagedRoute()` returns `null` and the assertion on the route statistics fails with "Failed to get managed route statistics." For Quarkus, add:

```xml
<dependency>
    <groupId>org.apache.camel.quarkus</groupId>
    <artifactId>camel-quarkus-management</artifactId>
</dependency>
```

For Spring Boot, add:

```xml
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-spring-boot-starter-management</artifactId>
</dependency>
```

You can find the complete Idempotent Receiver route and test in the [22-redis-integration example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/22-redis-integration/quarkus).

# Testing Circuit Breaker fallback paths

A Circuit Breaker wraps calls to an external service and provides a fallback when the service is unavailable. In Camel, you configure this with `circuitBreaker()` and `onFallback()`:

```java
from("kafka:eip.orders.processed?brokers={{kafka.brokers}}&groupId=managed-adapter-demo")
    .routeId("managed-adapter-circuit-breaker")
    .unmarshal().json()
    .log("Managed Adapter: checking inventory for order ${body[order_id]}")
    .circuitBreaker()
        .to("direct:inventory-check")
    .onFallback()
        .log("Circuit OPEN: routing order ${body[order_id]} to DLQ for retry")
        .marshal().json()
        .to("kafka:eip.orders.dlq?brokers={{kafka.brokers}}")
    .end()
    .log("Managed Adapter: order ${body[order_id]} processed successfully");

from("direct:inventory-check")
    .routeId("managed-adapter-inventory-check")
    .process(exchange -> {
        var body = exchange.getIn().getBody(java.util.Map.class);
        Number orderId = (Number) body.get("order_id");
        long id = orderId != null ? orderId.longValue() : 0;

        if (id % 3 == 0) {
            throw new RuntimeException(
                "Inventory service unavailable for order " + id);
        }

        body.put("inventory_status", "IN_STOCK");
        body.put("warehouse_location", "WH-CENTRAL-02");
    })
    .log("Inventory check passed for order ${body[order_id]}")
    .marshal().json()
    .to("kafka:eip.orders.inventory-checked?brokers={{kafka.brokers}}");
```

This route consumes orders from `eip.orders.processed`, wraps the inventory check in a circuit breaker, and routes to the DLQ when the check fails. 
The simulated inventory service fails for every order whose `order_id` is divisible by 3.

## Testing the happy path

When the inventory check succeeds, the order arrives on the `inventory-checked` topic with enriched fields:

```java
@Test
public void shouldProcessOrderAndCheckInventory() {
    t.given(
        createVariables()
            .variable("id", "1")
            .variable("amount", 200)
    );

    t.given(waitForCamelRouteStarted("managed-adapter-circuit-breaker", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.processed")
            .message()
            .body(Resources.create("templates/order.json"))
            .header(KafkaMessageHeaders.MESSAGE_KEY, "ADAPTER-${id}")
    );

    t.then(
        receive()
            .endpoint("kafka:eip.orders.inventory-checked?consumerGroup=citrus-inventory-checked-group")
            .message()
            .body("""
            {
              "order_id": ${id},
              "customer_id": "CUST-001",
              "item_sku": "SKU-TEST-1",
              "quantity": 1,
              "amount": ${amount},
              "inventory_status": "IN_STOCK",
              "warehouse_location": "WH-CENTRAL-02"
            }
            """)
    );
}
```

Notice the deterministic `id` value: `"1"`, not `citrus:randomNumber(4)`. 
The route's behavior depends on whether `order_id % 3 == 0`, so we need to control the exact value. 
Using `1` ensures the inventory check passes.

## Testing the fallback path

When the inventory check fails, the circuit breaker's `onFallback()` routes the message to the DLQ:

```java
@Test
public void shouldRouteToDeadLetterWhenCircuitBreaks() {
    t.given(
        createVariables()
            .variable("id", "3")
            .variable("amount", 300)
    );

    t.given(waitForCamelRouteStarted("managed-adapter-circuit-breaker", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.processed")
            .message()
            .body(Resources.create("templates/order.json"))
            .header(KafkaMessageHeaders.MESSAGE_KEY, "ADAPTER-DLQ-${id}")
    );

    t.then(
        receive()
            .endpoint("kafka:eip.orders.dlq?consumerGroup=citrus-dlq-group")
            .message()
            .body("""
            {
              "order_id": ${id},
              "customer_id": "CUST-003",
              "item_sku": "SKU-TEST-3",
              "quantity": 1,
              "amount": ${amount}
            }
            """)
    );
}
```

Here `id=3` — divisible by 3, triggering the failure. 
The `onFallback()` block marshals the original body to JSON and sends it to the DLQ. 
The test verifies that the DLQ message contains the original order fields without the enrichment fields (`inventory_status`, `warehouse_location`) that would only be present on a successful path.

## A common misconception about onFallback

It is worth clarifying: `onFallback()` fires on *any* exception thrown inside the `circuitBreaker()` block, not only when the circuit is in the "open" state. 
The first time the inventory check fails, the fallback executes immediately — the circuit does not need to have tripped from prior failures. 
This makes testing straightforward: you just need one message that triggers the exception. 
You do not need to send enough failing messages to trip the circuit first.

You can find the complete Circuit Breaker route and test in the [18-testing-management example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/18-testing-management/quarkus).

# Wrapping up

Testing reliability patterns means testing failure — deliberately, systematically, and reproducibly. The techniques we covered work together:

- **Dead Letter Channels**: use deterministic IDs that trigger exceptions, `repeatOnError` to poll the DLQ, and assert only on original message fields.
- **Idempotent Receivers**: use ordered tests to seed and re-send the same event, and `assertProcessedExchanges` to verify the duplicate handler.
- **Circuit Breakers**: use deterministic IDs that trigger fallback paths, and verify that the fallback output matches the pre-failure message.
- **Route testability**: extract inline branches into named `direct:` routes so every code path is visible to MBean-based assertions by accessing the route statistics and the number of completed exchanges.

You can explore all the reliability pattern examples in the [eip-with-camel repository](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing), specifically the [05-reliability](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/05-reliability), [22-redis-integration](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/22-redis-integration), and [18-testing-management](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/18-testing-management) examples.

Give it a try, and let us know what you think!
