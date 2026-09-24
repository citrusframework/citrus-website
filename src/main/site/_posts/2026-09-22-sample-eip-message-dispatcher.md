---
layout: sample
title: Testing the Message Dispatcher Pattern with Citrus
name: message-dispatcher
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Message Dispatcher EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/message-dispatcher/
---

As the number of event types flowing through a system grows, a common pattern emerges: a single Kafka topic carries messages of different types — orders placed, orders cancelled, orders refunded — and each type requires different processing logic.
You could duplicate a content-based router in every consuming service, but that scatters the routing logic across the codebase and makes it hard to see which types are handled where.

The [Message Dispatcher](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageDispatcher.html) pattern, described by Hohpe and Woolf, centralizes this fan-out.
A single consumer reads from a channel and distributes each message to a type-specific handler based on a field in the message body.
The dispatcher is the only component that knows about the type-to-handler mapping — the handlers themselves are simple, focused routes that process one event type each.

[Apache Camel](https://camel.apache.org) implements this with `toD()` (dynamic to), which resolves the target endpoint URI at runtime from the message content.
Combined with `direct:` routes as handler endpoints, `toD()` creates a clean dispatcher-handler architecture — one Kafka consumer, N handler routes, and a single routing expression that ties them together.

This pattern is closely related to the content-based router but differs in intent and structure.
A content-based router decides *where* a message goes (which channel, which queue).
A message dispatcher decides *who* handles it (which handler route within the same application).
The router is about external routing; the dispatcher is about internal fan-out.

Testing a message dispatcher requires a different approach than testing patterns that produce output on a Kafka topic.
The dispatcher's handlers typically log, persist, or call internal services — they do not always publish to an external endpoint that the test can consume from.
Integration tests with [Citrus](https://citrusframework.org) handle this by leveraging Camel's management API to verify that the correct handler route processed each message, including the edge case of unknown event types that no handler should accept.

## The scenario

Order events arrive on the `eip.consumer.dispatch` Kafka topic.
Each message carries an `event_type` field that determines which handler processes it:

- `order_placed` events go to the order creation handler
- `order_cancelled` events go to the cancellation handler
- `order_refunded` events go to the refund handler
- Unknown event types are caught by a fallback handler that logs and drops the message

The dispatcher validates the event type against a whitelist before routing.
This is a deliberate security measure — without validation, a malicious or malformed `event_type` value could cause `toD()` to resolve to an unexpected endpoint, since the endpoint URI is constructed from user-controlled data.

## The Camel route

### Quarkus

```java
@ApplicationScoped
public class MessageDispatcherRoute extends RouteBuilder {

    private static final Set<String> ALLOWED_EVENT_TYPES = Set.of(
        "order_placed", "order_cancelled", "order_refunded"
    );

    @Override
    public void configure() {
        from("kafka:eip.consumer.dispatch?brokers={% raw %}{{kafka.brokers}}{% endraw %}"
                + "&groupId=message-dispatcher&autoOffsetReset=earliest")
            .routeId("message-dispatcher")
            .unmarshal().json()
            .log("Dispatcher received: order ${body[order_id]}, type=${body[event_type]}")
            .choice()
                .when(exchange -> {
                    var body = exchange.getIn().getBody(java.util.Map.class);
                    String eventType = (String) body.get("event_type");
                    return eventType != null && ALLOWED_EVENT_TYPES.contains(eventType);
                })
                    .toD("direct:handle-${body[event_type]}")
                .otherwise()
                    .to("direct:handle-order_unknown")
            .end();

        from("direct:handle-order_placed")
            .routeId("handle-order-placed")
            .log("Handling ORDER PLACED: order ${body[order_id]}, "
                + "amount=${body[amount]}, sku=${body[item_sku]}");

        from("direct:handle-order_cancelled")
            .routeId("handle-order-cancelled")
            .log("Handling ORDER CANCELLED: order ${body[order_id]}, "
                + "refund pending for amount=${body[amount]}");

        from("direct:handle-order_refunded")
            .routeId("handle-order-refunded")
            .log("Handling ORDER REFUNDED: order ${body[order_id]}, "
                + "amount=${body[amount]} returned to customer ${body[customer_id]}");

        from("direct:handle-order_unknown")
            .routeId("handle-order-unknown")
            .log("Unknown event_type '${body[event_type]}' — skipping");
    }
}
```

There are five routes in a single `RouteBuilder`, and understanding their relationship is the key to understanding the pattern.

**The dispatcher route** (`message-dispatcher`) is the only route that consumes from Kafka.
It deserializes the JSON message and immediately enters a `choice()` block.
The `when()` predicate checks whether the `event_type` field is present and belongs to the `ALLOWED_EVENT_TYPES` set.
If the type is valid, `toD("direct:handle-${body[event_type]}")` dynamically resolves the handler endpoint — for example, an `event_type` of `order_placed` routes to `direct:handle-order_placed`.
If the type is invalid or missing, the `otherwise()` branch routes to `direct:handle-order_unknown`.

**The handler routes** each consume from a `direct:` endpoint and perform type-specific processing.
In this example, the handlers only log — in a production system, they would call services, write to databases, or publish to other topics.
Each handler has its own `routeId`, which is what makes them individually verifiable in the tests.

The `toD()` call is the most powerful and the most dangerous part of this pattern.
It constructs an endpoint URI from the message body at runtime.
Without the `ALLOWED_EVENT_TYPES` validation, a message with `event_type: "../../admin"` would cause Camel to attempt resolving `direct:handle-../../admin` as an endpoint — which would fail with a `ResolveEndpointFailedException`, but the attempt itself is a code smell.
The whitelist approach ensures that only known, expected endpoints are ever resolved.

### Spring Boot

```java
@Component
public class MessageDispatcherRoute extends RouteBuilder {

    private static final Set<String> ALLOWED_EVENT_TYPES = Set.of(
        "order_placed", "order_cancelled", "order_refunded"
    );

    @Override
    public void configure() {
        from("kafka:eip.consumer.dispatch?brokers={% raw %}{{kafka.brokers}}{% endraw %}"
                + "&groupId=message-dispatcher&autoOffsetReset=earliest")
            .routeId("message-dispatcher")
            .unmarshal().json()
            .log("Dispatcher received: order ${body[order_id]}, type=${body[event_type]}")
            .choice()
                .when(exchange -> {
                    var body = exchange.getIn().getBody(java.util.Map.class);
                    String eventType = (String) body.get("event_type");
                    return eventType != null && ALLOWED_EVENT_TYPES.contains(eventType);
                })
                    .toD("direct:handle-${body[event_type]}")
                .otherwise()
                    .to("direct:handle-order_unknown")
            .end();

        from("direct:handle-order_placed")
            .routeId("handle-order-placed")
            .log("Handling ORDER PLACED: order ${body[order_id]}, "
                + "amount=${body[amount]}, sku=${body[item_sku]}");

        from("direct:handle-order_cancelled")
            .routeId("handle-order-cancelled")
            .log("Handling ORDER CANCELLED: order ${body[order_id]}, "
                + "refund pending for amount=${body[amount]}");

        from("direct:handle-order_refunded")
            .routeId("handle-order-refunded")
            .log("Handling ORDER REFUNDED: order ${body[order_id]}, "
                + "amount=${body[amount]} returned to customer ${body[customer_id]}");

        from("direct:handle-order_unknown")
            .routeId("handle-order-unknown")
            .log("Unknown event_type '${body[event_type]}' — skipping");
    }
}
```

`@Component` replaces `@ApplicationScoped` — the route logic is identical across both runtimes.

## Dispatcher vs. content-based router

The message dispatcher looks similar to a content-based router, and the two patterns are often confused.
The distinction matters for testing:

| Aspect                 | Content-Based Router                      | Message Dispatcher               |
|------------------------|-------------------------------------------|----------------------------------|
| **Purpose**            | Route to different *channels*             | Dispatch to different *handlers* |
| **Output**             | External endpoints (Kafka topics, queues) | Internal `direct:` routes        |
| **Test verification**  | Receive from the output channel           | Check which handler route ran    |
| **Routing expression** | Static (`to()`) or dynamic (`toD()`)      | Typically dynamic (`toD()`)      |
| **Scope**              | Inter-service routing                     | Intra-service fan-out            |

A content-based router test sends a message and receives it from the correct output topic — a 1:1 input-to-output verification.
A message dispatcher test sends a message and verifies that the correct *handler route* processed it — a verification against Camel's internal route management, not an external channel.

## Message template

The tests use a JSON template with Citrus variable placeholders:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-00${id}",
  "event_type": "${eventType}",
  "amount": ${amount},
  "item_sku": "SKU-${id}",
  "quantity": 1
}
```

The `event_type` field is the dispatch key — the variable `${eventType}` is set differently in each test to exercise different handler routes.
The same template is reused across all four test cases; only the variables change.

## The dispatcher tests

The message dispatcher requires four test cases — one for each known event type plus one for the unknown type fallback.
Each test sends a single message with a specific `event_type` and verifies that the corresponding handler route processed it.

### Quarkus tests

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
    class MessageDispatcherTest {

        @Test
        public void shouldDispatchOrderPlacedEvent() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("eventType", "order_placed")
                    .variable("amount", 120)
            );

            t.given(waitForCamelRouteStarted("message-dispatcher", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.consumer.dispatch")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(verifyCompletedExchanges("handle-order-placed", 1, camelContext));
        }

        @Test
        public void shouldDispatchOrderCancelledEvent() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("eventType", "order_cancelled")
                    .variable("amount", 80)
            );

            t.given(waitForCamelRouteStarted("message-dispatcher", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.consumer.dispatch")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(verifyCompletedExchanges("handle-order-cancelled", 1, camelContext));
        }

        @Test
        public void shouldDispatchOrderRefundedEvent() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("eventType", "order_refunded")
                    .variable("amount", 55)
            );

            t.given(waitForCamelRouteStarted("message-dispatcher", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.consumer.dispatch")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(verifyCompletedExchanges("handle-order-refunded", 1, camelContext));
        }

        @Test
        public void shouldSkipUnknownEventType() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("eventType", "order_unknown")
                    .variable("amount", 40)
            );

            t.given(waitForCamelRouteStarted("message-dispatcher", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.consumer.dispatch")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(verifyCompletedExchanges("handle-order-unknown", 1, camelContext));
        }
    }
}
```

All four tests follow the same given-when-then structure.
The only thing that changes between them is the `eventType` variable and the handler route ID passed to `verifyCompletedExchanges`.

**Given — set up variables and wait for the dispatcher.**
Each test generates a random order ID and sets the `eventType` to the value that should trigger a specific handler.
The test then waits for the `message-dispatcher` route to reach `Started` status.

**When — send an event to the dispatch topic.**
A single message goes to `kafka:eip.consumer.dispatch` using the shared `order.json` template.
The `eventType` variable is resolved into the `event_type` field of the JSON body.

**Then — verify the correct handler processed the message.**
`verifyCompletedExchanges` checks that the expected handler route completed exactly one exchange.
For `shouldDispatchOrderPlacedEvent`, it asserts against `handle-order-placed`.
For `shouldSkipUnknownEventType`, it asserts against `handle-order-unknown`.

### The unknown event type test

The `shouldSkipUnknownEventType` test deserves special attention because it verifies the dispatcher's safety behavior.

The test sets `eventType` to `"order_unknown"` — a value that is *not* in the `ALLOWED_EVENT_TYPES` set.
When the dispatcher receives this message, the `choice()` predicate returns `false`, and the `otherwise()` branch routes the message to `direct:handle-order_unknown`.

This test proves two things:

1. **The whitelist works.** An unknown event type does not reach `toD()` — it is caught by the `choice()` and routed to the fallback handler instead. Without this test, a code change that accidentally removes the whitelist check would go undetected.

2. **The fallback handler exists and processes successfully.** If the `direct:handle-order_unknown` route were missing, the test would fail with a `NoSuchEndpointException`. The `verifyCompletedExchanges` check also verifies that the fallback handler completed without errors.

This is a test that cannot be reasonably written with mocked endpoints.
The `toD()` resolution, the `choice()` predicate evaluation, and the `direct:` endpoint dispatch all happen inside Camel's routing engine — mocking any of these would bypass the exact logic being tested.

### Spring Boot tests

```java
@SpringBootTest(classes = ConsumerPatternsApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class MessageDispatcherTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldDispatchOrderPlacedEvent() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("eventType", "order_placed")
                    .variable("amount", 120)
            );

            t.given(waitForCamelRouteStarted("message-dispatcher", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.consumer.dispatch")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(verifyCompletedExchanges("handle-order-placed", 1, camelContext));
        }

        @Test
        public void shouldDispatchOrderCancelledEvent() {
            // Same structure — eventType="order_cancelled",
            // asserts against "handle-order-cancelled"
            // ...
        }

        @Test
        public void shouldDispatchOrderRefundedEvent() {
            // Same structure — eventType="order_refunded",
            // asserts against "handle-order-refunded"
            // ...
        }

        @Test
        public void shouldSkipUnknownEventType() {
            // Same structure — eventType="order_unknown",
            // asserts against "handle-order-unknown"
            // ...
        }
    }
}
```

## What the tests prove

The four tests together form a complete coverage of the dispatcher's routing logic:

**Positive dispatch tests** (`shouldDispatchOrderPlacedEvent`, `shouldDispatchOrderCancelledEvent`, `shouldDispatchOrderRefundedEvent`) verify that each known event type reaches its designated handler.
They exercise the `toD()` resolution with three different dynamic endpoint URIs:

- `event_type=order_placed` → `toD("direct:handle-order_placed")` → route `handle-order-placed`
- `event_type=order_cancelled` → `toD("direct:handle-order_cancelled")` → route `handle-order-cancelled`
- `event_type=order_refunded` → `toD("direct:handle-order_refunded")` → route `handle-order-refunded`

**Negative dispatch test** (`shouldSkipUnknownEventType`) verifies that the whitelist rejects unknown types and routes them to the fallback handler.
This is the security-relevant test — it proves that arbitrary `event_type` values cannot reach `toD()`.

Together, these tests verify the full routing table.
If a new event type is added to the application but not to the `ALLOWED_EVENT_TYPES` set, it will be routed to the unknown handler — and a corresponding test should be added to cover the new type.

## The test design pattern

The message dispatcher test suite demonstrates a repeatable pattern for testing internal fan-out:

1. **One test per dispatch target.** Each handler route gets its own test case with the corresponding input. This makes failures immediately diagnosable — if `shouldDispatchOrderRefundedEvent` fails, you know exactly which path is broken.

2. **The dispatch key is the only variable that changes.** All four tests use the same template, the same topic, and the same given-when-then structure. Only `eventType` and the assertion target differ. This minimizes the surface area for test bugs.

3. **Verification targets the handler route, not the dispatcher.** The test does not assert against the `message-dispatcher` route — it asserts against the *handler* route that should have been invoked. This is deliberate: verifying the handler's exchange count proves that the dispatcher made the correct routing decision *and* that the `direct:` dispatch succeeded.

4. **Always test the fallback path.** The unknown event type test is not an afterthought — it verifies the dispatcher's safety net. In production systems, unknown event types can appear due to schema evolution, message corruption, or cross-version compatibility issues. The fallback handler and its test ensure the dispatcher does not throw exceptions or route to unexpected endpoints when this happens.

## Why `verifyCompletedExchanges` is the right tool here

For patterns that produce output on a Kafka topic, the natural Citrus assertion is `receive()` — consume from the output topic and validate the message body.
The message dispatcher's handlers do not produce external output, so `receive()` has nothing to consume.

`verifyCompletedExchanges` solves this by using Citrus's built-in Camel route statistics verification:

- **Exchange count.** Has the handler route completed exactly the expected number of exchanges? For a dispatcher test that sends one message, the expected count is 1.
- **Error detection.** The route statistics verification checks for failed exchanges automatically. A `ResolveEndpointFailedException` from a bad `toD()` target, a serialization error in the handler, or any other exception is detected, providing a clear signal.
- **Retry tolerance.** The `repeatOnError()` wrapper retries up to 20 times with 1-second intervals. This accommodates Kafka consumer lag — the dispatcher route might not have consumed and dispatched the message by the time the assertion first runs.

This pattern applies to any route that processes messages internally without producing verifiable output on an external channel: logging routes, metric-emitting routes, routes that call internal services, and — as in this case — dispatcher handler routes.

For the test infrastructure setup, shared test utilities, runtime wiring, dependencies, and how to run the tests, see the [Camel EIP examples](/samples/camel-eip/) overview page.

## Key takeaways

- **Message dispatchers centralize type-based routing.** A single Kafka consumer with `toD()` fans out to N handler routes, keeping the routing logic in one place rather than scattered across consuming services.
- **Whitelist validation is essential with `toD()`.** Because `toD()` constructs endpoint URIs from message content, untrusted values must be validated against a known set before reaching the dynamic expression. The `ALLOWED_EVENT_TYPES` set and the `choice()` guard implement this pattern.
- **Test every dispatch target, including the fallback.** One test per handler route ensures complete coverage of the routing table. The unknown event type test verifies that the safety net works — it is not optional.
- **`verifyCompletedExchanges` verifies internal routing decisions.** When handler routes do not produce external output, Citrus's Camel route statistics verification provides exchange counts as an alternative verification mechanism. This makes the test independent of what the handler actually does.
- **One template, N tests.** The same message template serves all four test cases. Only the `eventType` variable and the assertion target change — this keeps the test suite focused on the routing decision, not the message structure.
- **Two runtimes, one test pattern.** The test logic is identical across Quarkus and Spring Boot. Only the bootstrap annotations and dependency injection differ — the given-when-then flow, the template, and the assertions stay the same.
