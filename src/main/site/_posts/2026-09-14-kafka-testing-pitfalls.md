---
layout: post
title: Kafka Testing Pitfalls in Camel Integration Tests
short-title: Kafka Testing Pitfalls
author: Christoph Deppisch
github: christophd
categories: [blog]
---

Kafka adds a unique set of challenges to integration testing. 
Offsets, consumer groups, header serialization, and shared topics introduce failure modes that do not exist with simpler transports like direct or SEDA endpoints.

This post covers Kafka-specific testing pitfalls collected from real-world [Apache Camel](https://camel.apache.org) integration projects, each with a concrete fix.
For general Camel testing pitfalls — route testability, missing marshal steps, timer interference, and more — see the companion post [Camel Integration Testing Pitfalls](/news/2026/09/14/camel-testing-pitfalls/).

![Citrus](/img/assets/camel-testing-pitfalls/featured.png){:width="700px" .center-image}
*AI generated with Google Gemini*

# Pitfall: Kafka offset and racing conditions

Multi-hop flows are common in Camel: route1 consumes messages from topic A, processes the message and produces an outcome to topic B. 
The test sends to topic A and expects to receive the output from topic B. But there is a timing problem.

## The problem

When the test sends the initial message to topic A the consumer on topic B may not have established its offset yet. 
So the Camel route processing and the verifying test consumer enter a racing condition. 
Racing conditions are always bad in terms of unstable tests because the success or failure is dependent on the performance of the machine that runs the tests.
By the time Camel produces the message on topic B and the message has already been committed at an offset the consumer may skip this message because `auto.offset.reset=latest` is the default for new consumer groups in Citrus.

The test passes sometimes and fails sometimes (with message timeout error), depending on how fast the Camel route processes the events and how fast the test consumers initialize.

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

    t.given(waitForCamelRouteStarted("order-processing", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
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

## The fix

Basically we run into these timing problems because the test both produces the initial event and consumes the triggered output event for verification purpose.
So the test is producer and consumer at the same time with the Camel route processing in the middle.

The fix to prevent timing issues is to use `fork(true)` when sending the 1st event to the Camel route.

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

    t.given(waitForCamelRouteStarted("order-processing", camelContext));

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

The `fork(true)` on the send action forces Citrus to perform the Kafka send operation and the subsequent receive action concurrently. 
Citrus sends the message and immediately starts the `receive` action listening for the event on the output topic.

The fork enabled `send` avoids the situation where the test sends the message, waits for the Kafka acknowledgement, then starts listening, and potentially misses the processed message if the Camel route finishes before the Citrus consumer is ready.
This can lead to flaky tests where the timing condition may succeed or fail in an unpredictable way.

You can find the complete code and test in the [05-reliability example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/05-reliability/quarkus).

# Pitfall: Cross-test interference on shared Kafka topics

Multiple tests often produce/consume event on the same Kafka topics. 
When one test sends data to `eip.orders.placed`, it might interfere with other tests also consuming from that topic.

Also, multiple tests may try to verify output messages on the same topic for different use cases. 
This is the situation where running a single test is successful but running the same test together with other tests in a test suite fails all of a sudden.
The reason is that multiple tests running in the same test suite creates an interference where one test steals messages from another test.

## The problem

A test that sends an order with `amount=250` to `eip.orders.placed` and expects to receive it on `eip.orders.domestic` will work. 
But it also interferes with other tests that use the same topic. 
A later test for the Message Filter might pick up this message instead of its own test data, causing a false pass or an unexpected failure.

## The fix

Multiple strategies work together to fix this situation:

**Design test data to be inert for unrelated routes.** Be sure to use different test data for different tests. 
Always using the same test customer or the same order id may interfere with other tests. 
Citrus provides the concept of test variables where each test can use unique identifiers and data.

```java
t.given(createVariables()
    .variable("id", "citrus:randomNumber(4)")
    .variable("amount", 75.00)   // below filter threshold
    .variable("country", "US")
    .variable("hazmat", false)
);
```

**Use unique consumer groups on every receive action.** Never reuse consumer group names across tests. 
This ensures that another test does not consume messages that were supposed for the current test because both tests have been using the very same consumer group.

```java
// Test A
receive().endpoint("kafka:eip.orders.domestic?consumerGroup=citrus-domestic-group")

// Test B — different consumer group, even though same topic
receive().endpoint("kafka:eip.orders.domestic?consumerGroup=citrus-domestic-other-group")
```

_IMPORTANT_: Sometimes using the very same consumer group (within the same test) may be mandatory. 
For instance when a test verifies idempotent message processing by sending duplicates to a topic and consuming from the same topic multiple times to check that only one event has been produced.

**Use message keys or headers to correlate send and receive.** When inert data is not possible, use a `KafkaMessageFilter` to select only messages matching your test's identifier:

```java
t.then(
    receive()
        .selector(KafkaMessageFilter.kafkaMessageFilter()
                .eventLookbackWindow(Duration.ofSeconds(10))
                .kafkaMessageSelector(kafkaHeaderEquals("order-id", "${id}"))
                .build())
        .endpoint("kafka:eip.orders.processed?consumerGroup=citrus-processed-group")
        .message()
        .body(Resources.create("templates/order.json"))
);
```

The `KafkaMessageFilter` scans messages within the `lookback` window and selects only those whose `order-id` header matches the test variable. 
Messages from other tests or routes are ignored.

**Review the Kafka auto offset setting** Each consumer on a Kafka topic uses the offset setting to define where in the message history to start consuming messages. 

Kafka's `auto.offset.reset=earliest` setting means the consumer reads from the beginning of the topic. 
If a prior test class wrote a message to that topic, the `receive()` assertion finds it — even though the current test did not produce any output.

Kafka's `auto.offset.reset=latest` setting skips all messages that arrived prior to the consumer starting to listen. 
This may lead to racing conditions where the consumer misses an already commited message from the same test. 
The fork option on the send operation in Citrus may help to solve this racing condition.

# Pitfall: Different value types in Kafka headers

When Citrus sends messages through Kafka using `camel().send()` with `CamelEndpointBuilder`, Kafka serializes header values to byte arrays.
The arbitrary `send()` action in Citrus uses String serializer for header values, by default.

This means based on the used header value serialization strategy Kafka headers may have different types. 

On the Camel route consumer side, `exchange.getIn().getHeader("myHeader")` returns `byte[]`, not `String`. 
Calling `Long.parseLong(header.toString())` produces `NumberFormatException` because `byte[].toString()` returns something like `[B@3a1c9d4f`.

## The problem

A Camel route reads a timestamp value from a Kafka header and assumes a very specific value type for this header:

```java
// Fails when header value is a byte array
Object value = exchange.getIn().getHeader("orderTimestamp");
long orderTimestamp = Long.parseLong(value.toString());
```

Depending on how the Kafka message was sent to the topic the message header value type might differ based on the header value serialization strategy.
When the message was sent via `CamelEndpointBuilder` in a Citrus test, `orderTimestamp` is a `byte[]` array and `value.toString()` is not parseable.

## The fix

The fix can be done on the Citrus test or on the Camel route.
The Citrus test should make sure to use the proper Kafka message header value serialization strategy that matches the Camel route logic.
In this case this would be a String value serializer as the Camel route expects the header to of type String.
You can configure the serializer/deserializer strategy for Kafka endpoints in Citrus and Camel via the endpoint configuration.

On the other hand the Camel route can use a more robust processing logic by handling both header value types with an `instanceof` check in the route:

```java
.process(exchange -> {
    long orderTimestamp = 0;
    Object ts = exchange.getIn().getHeader("orderTimestamp");
    if (ts instanceof byte[]) {
        orderTimestamp = Long.parseLong(new String((byte[]) ts));
    } else if (ts != null) {
        orderTimestamp = Long.parseLong(ts.toString());
    }
    exchange.getIn().setHeader("messageAge",
        System.currentTimeMillis() - orderTimestamp);
})
```

On the test side, send numeric headers as strings to match what a Kafka producer would normally send:

```java
t.when(
    camel()
        .send()
        .endpoint(CamelSupport.camel().endpoints()
                .kafka("eip.orders.accepted")
                .brokers("localhost:9092")::getRawUri)
        .message()
        .body(Resources.create("templates/order.json"))
        .header("orderTimestamp", String.valueOf(System.currentTimeMillis()))
);
```

The `String.valueOf()` ensures the header is a string before Kafka serializes it. The route-side `instanceof` check handles both raw strings (from production Kafka producers) and byte arrays (from test CamelEndpointBuilder sends).

# A Kafka testing checklist

Here are all Kafka pitfalls as a quick-reference checklist:

- Use `fork(true)` on the initial `send()` in multi-hop Kafka flows to avoid racing conditions where the Camel route produces the output before the test consumer has established its offset.
- Use unique consumer groups per receive action, unique test data via test variables, and `KafkaMessageFilter` selectors to prevent cross-test interference on shared Kafka topics.
- Handle Kafka headers with consistent serialization strategies or support both `byte[]` and `String` value types in route processors with an `instanceof` check.
- Review the `auto.offset.reset` setting for each test consumer — `earliest` risks picking up leftover messages, `latest` risks missing already committed messages.

None of these pitfalls are specific to [Citrus](https://citrusframework.org) — they apply to Kafka integration testing in general. But Citrus gives you the tools and capabilities to fix every one of them.

You can explore all the testing patterns and pitfall fixes in the [eip-with-camel repository](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing).

Give it a try, and let us know what you think!
