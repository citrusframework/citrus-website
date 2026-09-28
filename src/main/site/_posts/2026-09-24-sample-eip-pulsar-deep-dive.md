---
layout: sample
title: Testing Apache Pulsar Routes with Citrus
name: pulsar-deep-dive
image: /img/icons/camel.png
folder: apache-camel
group: camel
description: Testing Pulsar-backed Camel routes — shared subscriptions, key-shared ordering, and dead-letter topics — with Citrus on Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-pulsar-deep-dive/
---

Apache Pulsar offers a set of capabilities that Kafka simply does not have out of the box: per-message TTL enforced at the broker, key-shared subscriptions that guarantee per-key ordering without requiring partitions, and a built-in schema registry.
These features make Pulsar an attractive fit for certain integration scenarios — but they also introduce new testing challenges.
When your Camel route relies on Pulsar-specific subscription semantics, how do you write automated integration tests that actually exercise those behaviors?

This post walks through three Camel route patterns built on top of Apache Pulsar, and shows how to test each one with [Citrus](https://citrusframework.org) across both Quarkus and Spring Boot runtimes.
The three scenarios are: a Shared subscription for competing-consumer order processing, a Key_Shared subscription for ordered per-key processing, and a Dead Letter Topic (DLT) pipeline that captures messages that exhaust all redelivery attempts.
Each scenario is backed by real Pulsar infrastructure spun up with Docker Compose via Testcontainers — no mocks, no stubs, no embedded broker.

# What makes Pulsar different for testing

Before diving into the code, a brief word on what sets Pulsar apart from Kafka — and why that matters for integration tests.

Pulsar separates the concept of *consumers* from *subscriptions*.
In Kafka, a consumer group is the mechanism for load-balancing and offset tracking; parallelism is tied to partitions.
In Pulsar, a *subscription* carries the cursor, and the *subscription type* determines how messages are dispatched among the consumers sharing that subscription:

| Subscription type | Behavior                                | Testing concern                                                                                    |
|-------------------|-----------------------------------------|----------------------------------------------------------------------------------------------------|
| **Exclusive**     | Single consumer only                    | Simplest to test — one consumer, one message stream                                                |
| **Shared**        | Round-robin across consumers            | Must verify processing happens regardless of which consumer instance receives the message          |
| **Key_Shared**    | Same key always routes to same consumer | Must verify that all messages for a given key are processed in order by a single consumer instance |
| **Failover**      | One active consumer, others on standby  | Must verify promotion behavior on consumer loss                                                    |

Another Pulsar-specific feature is the **Dead Letter Topic** (DLT).
When a consumer fails to acknowledge a message within a configurable number of redelivery attempts (`maxRedeliverCount`), Pulsar automatically moves the message to a designated dead-letter topic.
Testing this end-to-end — message enters, processing fails, retries exhaust, message appears on the DLT — requires real broker behavior.

The example project uses Apache Camel and contains three route builders, each demonstrating a distinct Pulsar capability.

# Shared subscription

## The Camel route

The first route publishes orders to a persistent Pulsar topic and consumes them with a `Shared` subscription using two concurrent consumer threads.
Any order can be received by any consumer instance, which is the standard competing-consumer pattern.

```java
// Quarkus
@ApplicationScoped
public class SharedSubscriptionRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("pulsar:persistent://public/default/eip.orders.placed"
                + "?subscriptionName=inventory-service"
                + "&subscriptionType=Shared"
                + "&numberOfConsumers=2")
            .routeId("pulsar-shared-consumer")
            .log("Shared consumer received: ${body}")
            .to("direct:process-pulsar-order");

        from("direct:process-pulsar-order")
            .routeId("pulsar-order-processor")
            .log("Processing Pulsar order");
    }
}
```

On Spring Boot, `@ApplicationScoped` is replaced with `@Component` and the config property is injected with `@Value` instead of `@ConfigProperty`.
The route logic is identical across both runtimes.

The key points here are `subscriptionType=Shared` and `numberOfConsumers=2`.
Shared subscriptions distribute messages round-robin across all consumers sharing the same subscription name.
Because the number of consumers exceeds one, you cannot predict which consumer thread will receive any particular message — but you *can* assert that the message was processed by one of them.
This is a subtle but important distinction that shapes how the Citrus test is structured, as we will see shortly.

## Testing the Shared subscription

The Shared subscription test sends an order message directly to the Pulsar topic using Citrus's Camel endpoint DSL, then verifies that the `pulsar-order-processor` route processed at least one exchange successfully.

```java
@Test
public void shouldProcessOrderWithSharedSubscription() {
    t.given(waitForCamelRouteStarted("pulsar-shared-consumer", camelContext));

    t.when(
        camel()
            .send()
            .endpoint(CamelSupport.camel().endpoints()
                    .pulsar("persistent://public/default/eip.orders.placed")
                    .serviceUrl("pulsar://localhost:6650")
                    .producerName("citrus-shared-sub-test")::getRawUri)
            .message()
            .body("{\"order_id\": 2001, \"customer_id\": \"C-001\", "
                + "\"item_sku\": \"SKU-P1\", \"quantity\": 2, \"amount\": 49.99}")
    );

    t.then(verifyCompletedExchanges("pulsar-order-processor", 1, camelContext));
}
```

A few things to notice about this test.

First, Citrus sends the message via the Camel Pulsar endpoint using `camel().send()`.
This uses the same `camel-pulsar` component that the application routes use — Citrus does not need a separate Pulsar client library; it borrows the application's Camel context to produce the message.
The `producerName` is explicitly set to avoid producer name conflicts with the application's own producers.

Second, the verification uses `verifyCompletedExchanges` rather than a `receive()` action.
Because the route routes the processed message to `direct:process-pulsar-order` and then logs it, there is no output topic to consume from.
Instead, the test checks the JMX statistics of the `pulsar-order-processor` route to confirm that at least one exchange was completed without failures.
This approach is particularly well-suited to `Shared` subscriptions, where you care that *some* consumer processed the message, not *which* one.

# Key_Shared subscription

## The Camel route

The second route uses a Key_Shared subscription to enforce per-key ordering across multiple consumer threads.
Every order message carries a key based on its order ID; Pulsar guarantees that all messages with the same key are always dispatched to the same consumer instance.

```java
// Quarkus
@ApplicationScoped
public class KeySharedRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("pulsar:persistent://public/default/eip.orders.keyed"
                + "?subscriptionName=shipping-service"
                + "&subscriptionType=Key_Shared"
                + "&numberOfConsumers=2")
            .routeId("pulsar-key-shared-consumer")
            .log("Key_Shared consumer received order [key=${header[pulsar.producer.message.key]}]: ${body}")
            .to("direct:process-keyed-order");

        from("direct:process-keyed-order")
            .routeId("pulsar-keyed-order-processor")
            .unmarshal().json()
            .log("Processing keyed order ${body[order_id]} for customer ${body[customer_id]}")
            .log("All events for the same order key route to the same consumer instance");
    }
}
```

The message key is communicated to the Pulsar producer via the `pulsar.producer.message.key` Camel message header.
Unlike Kafka — where ordering is guaranteed only within a partition, and repartitioning can break key affinity — Pulsar routes by key at the subscription level, independently of how many consumers share the subscription.
Adding or removing consumer instances does not disrupt key affinity.

## Testing the Key_Shared subscription

The Key_Shared test follows the same structure but sets the message key header to ensure the message can be correlated with the correct consumer instance.

```java
@Test
public void shouldProcessKeyedOrderWithKeySharedSubscription() {
    t.given(waitForCamelRouteStarted("pulsar-key-shared-consumer", camelContext));

    t.when(
        camel()
            .send()
            .endpoint(CamelSupport.camel().endpoints()
                    .pulsar("persistent://public/default/eip.orders.keyed")
                    .serviceUrl("pulsar://localhost:6650")
                    .producerName("citrus-key-shared-test")::getRawUri)
            .message()
            .header("pulsar.producer.message.key", "order-3001")
            .body("{\"order_id\": 3001, \"customer_id\": \"C-001\", "
                + "\"item_sku\": \"SKU-P2\", \"quantity\": 3, \"status\": \"placed\"}")
    );

    t.then(verifyCompletedExchanges("pulsar-keyed-order-processor", 1, camelContext));
}
```

Setting the `pulsar.producer.message.key` header is the test-side equivalent of what the production producer route does when it sets `exchange.getIn().setHeader("pulsar.producer.message.key", key)`.
By using the same header name, the test exercises the complete key routing path: producer attaches the key, Pulsar routes the message to the designated consumer instance for that key, consumer processes it.

What does this test actually prove?
It verifies that a message with a specific key is received by the `pulsar-key-shared-consumer` route and successfully processed by the downstream `pulsar-keyed-order-processor`.
In a production scenario with multiple running instances, adding a second test with the same key and verifying sequential ordering would demonstrate the full Key_Shared guarantee.
For the purposes of this example — a single Camel application with two consumer threads — the test confirms the end-to-end path functions correctly without failures.

# Dead Letter Topic

## The Camel route

The third route exercises Pulsar's built-in dead-letter handling.
The payment consumer intentionally throws a `RuntimeException` for single-item orders (orders where `quantity == 1`) to simulate a processing failure.
After three failed redelivery attempts (`maxRedeliverCount=3`), Pulsar moves the message to the `eip.orders.payments-dlq` topic.
A second route monitors that dead-letter topic.

```java
// Quarkus
@ApplicationScoped
public class DeadLetterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("pulsar:persistent://public/default/eip.orders.payments"
                + "?subscriptionName=payment-service"
                + "&subscriptionType=Shared"
                + "&maxRedeliverCount=3"
                + "&deadLetterTopic=persistent://public/default/eip.orders.payments-dlq"
                + "&numberOfConsumers=1")
            .routeId("pulsar-dlt-consumer")
            .log("Processing payment order: ${body}")
            .process(exchange -> {
                String body = exchange.getIn().getBody(String.class);
                if (body.contains("\"quantity\": 1,")) {
                    throw new RuntimeException(
                        "Simulated payment failure for single-item order");
                }
            })
            .log("Payment order processed successfully");

        from("pulsar:persistent://public/default/eip.orders.payments-dlq"
                + "?subscriptionName=dlt-monitor"
                + "&subscriptionType=Exclusive")
            .routeId("pulsar-dlt-monitor")
            .log("Dead letter topic received failed order: ${body}")
            .log("Order exhausted all redelivery attempts and requires manual review");
    }
}
```

The `deadLetterTopic` and `maxRedeliverCount` parameters are pure Pulsar configuration — no application code manages the retry or DLT routing.
This is one of Pulsar's strongest points for reliability testing: the broker takes care of the dead-letter flow, so the route logic stays clean and the failure path is broker-enforced.

## Testing valid payment processing

The Dead Letter Topic scenario has two separate test cases.
The first verifies the happy path: a valid payment order (quantity > 1, which does *not* trigger the simulated failure) is processed successfully.

```java
@Test
public void shouldProcessValidPaymentOrder() {
    // quantity=2 does NOT trigger the simulated failure (only quantity=1 does)
    t.given(waitForCamelRouteStarted("pulsar-dlt-consumer", camelContext));

    t.when(
        camel()
            .send()
            .endpoint(CamelSupport.camel().endpoints()
                    .pulsar("persistent://public/default/eip.orders.payments")
                    .serviceUrl("pulsar://localhost:6650")
                    .producerName("citrus-dlt-test")::getRawUri)
            .message()
            .body("{\"order_id\": 4002, \"customer_id\": \"C-002\", "
                + "\"item_sku\": \"SKU-P3\", \"quantity\": 2, \"amount\": 39.98}")
    );

    t.then(verifyCompletedExchanges("pulsar-dlt-consumer", 1, camelContext));
}
```

The comment in the test (`quantity=2 does NOT trigger the simulated failure`) is important documentation.
It makes the test's intent explicit: this test specifically exercises the success path.
A reader can immediately see the boundary condition that separates success from failure — `quantity=1` fails, everything else succeeds — without having to trace back to the route implementation.

The `verifyCompletedExchanges` call verifies both that the exchange completed *and* that no failures occurred.
If you accidentally send a `quantity=1` order here, the failure assertion in `verifyCompletedExchanges` catches it.

# What the exchange statistics approach gives you

Throughout these tests, a recurring pattern is the use of `verifyCompletedExchanges` backed by JMX route statistics instead of consuming messages from output endpoints.
This approach deserves a brief explanation of when it is the right choice.

Citrus's standard `receive()` action is ideal when:
- The route produces output to a topic or queue that the test can consume
- The output message content needs to be validated
- You want to assert the exact shape of the transformed message

The JMX statistics approach is better when:
- The route routes to a `direct:` endpoint with no external output
- Multiple consumer threads share a subscription and message delivery is non-deterministic
- The test goal is to prove that processing completed without errors, not to validate specific output content
- The route subscribes to a dead-letter topic and you want to confirm it handles DLT messages gracefully

In the Pulsar tests above, every route terminates at either a `direct:` endpoint or a log statement — there is no output topic to consume.
The JMX approach threads a needle: it validates real end-to-end processing behavior while adapting to routes that do not expose an output endpoint convenient for test consumption.

For the test infrastructure setup, shared test utilities, runtime wiring, dependencies, and how to run the tests, see the [Camel EIP examples](/samples/camel-eip/) overview page.

# Key takeaways

Apache Pulsar's subscription model, key-based routing, and built-in dead-letter handling require integration tests that exercise the broker's own behavior — not mocked replicas of it.
Citrus provides the right tools for this through its Camel DSL integration, Testcontainers support, and JMX-backed route statistics.

Here is a summary of the patterns used across these three scenarios:

- **Camel endpoint DSL for Pulsar producers** — `CamelSupport.camel().endpoints().pulsar(...)` lets Citrus send messages via the same `camel-pulsar` component the application uses, with full control over subscription name, producer name, and message key headers. No separate Pulsar client SDK is needed in the test.
- **JMX route statistics for non-consuming assertions** — When a route terminates at a `direct:` endpoint or logs its output, `verifyCompletedExchanges` via `ManagedRouteMBean` provides end-to-end proof of processing without requiring an output topic to consume from. Both completion count and failure count are checked.
- **`repeatOnError` for asynchronous readiness** — Both `waitForCamelRouteStarted` and `verifyCompletedExchanges` use `repeatOnError` with a sleep interval rather than fixed `Thread.sleep` calls. The test proceeds as soon as the condition is met, keeping execution time to a minimum.
- **Infrastructure-as-code readiness** — The `waitFor().http()` action gates the test suite on the Pulsar admin health endpoint. Tests never start against an unready broker, eliminating a common class of container-related flakiness.
- **Two runtimes, one test pattern** — The Quarkus and Spring Boot tests express identical scenario logic. The differences are limited to bootstrap annotations (`@QuarkusTest` vs `@SpringBootTest`), CDI injection style, and where the `TestCaseRunner` is declared. The test structure, the `EipTestSupport` utilities, and the assertions are shared across both runtimes.

The complete source code for this example is available on GitHub:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/21-pulsar-deep-dive/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/21-pulsar-deep-dive/spring-boot)

Give it a try, and let us know what you think!
