---
layout: sample
title: Testing the Resequencer Pattern with Citrus
name: resequencer
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Resequencer EIP in Apache Camel with Citrus across Quarkus, Spring Boot, and YAML DSL
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/resequencer/
---

Messages arrive in order.
Or at least, that is the assumption most developers carry into their first integration project.
In practice, distributed systems rarely guarantee ordering.
Network retries, parallel consumers, partitioned queues, and upstream fan-out/fan-in patterns all conspire to deliver messages out of sequence.

Consider an order processing system that receives shipping instructions from multiple upstream services.
Each instruction carries a sequence number — 1 for the first step, 2 for the second, and so on.
A downstream fulfillment service expects these instructions in order so it can apply them incrementally.
When message 3 arrives before message 1, the fulfillment logic breaks.

The [Resequencer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Resequencer.html) pattern, described by Hohpe and Woolf, solves this by collecting messages in a buffer and re-emitting them in sequence order.
[Apache Camel](https://camel.apache.org) implements two variants: a *batch resequencer* that collects up to N messages (or waits a timeout) and then sorts the batch, and a *stream resequencer* that emits messages as soon as the next expected sequence number arrives.

Testing a resequencer is fundamentally different from testing stateless routing patterns like a content-based router or a message filter.
Those patterns make an immediate routing decision per message.
A resequencer, by contrast, *holds* messages — it accumulates a batch, waits for a timeout, and only then produces output.
This time-dependent, stateful behavior makes it a compelling candidate for automated integration testing with [Citrus](https://citrusframework.org).

## The Camel route

The example uses a batch resequencer that reads orders from a Kafka topic, collects them in a batch window, and re-emits them sorted by a `sequenceNumber` header.

### Quarkus

```java
@ApplicationScoped
public class ResequencerRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.sequenced?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=resequencer-demo")
            .routeId("batch-resequencer")
            .unmarshal().json()
            .log("Received out-of-order message seq=${header.sequenceNumber}")
            .resequence(header("sequenceNumber"))
                .batch()
                .size(10)
                .timeout(5000)
            .log("Resequenced message seq=${header.sequenceNumber}, order ${body[order_id]}")
            .marshal().json()
            .to("kafka:eip.orders.resequenced?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

The route unmarshals each incoming JSON message, then feeds it into the `resequence()` EIP using `header("sequenceNumber")` as the sort key.
The `.batch()` configuration tells Camel to collect up to 10 messages or wait 5 seconds (whichever comes first), sort the batch by the sequence number, and then forward each message in order to the output topic.

### Spring Boot

The Spring Boot variant is identical except for the CDI annotation:

```java
@Component
public class ResequencerRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.sequenced?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=resequencer-demo")
            .routeId("batch-resequencer")
            .unmarshal().json()
            .log("Received out-of-order message seq=${header.sequenceNumber}")
            .resequence(header("sequenceNumber"))
                .batch()
                .size(10)
                .timeout(5000)
            .log("Resequenced message seq=${header.sequenceNumber}, order ${body[order_id]}")
            .marshal().json()
            .to("kafka:eip.orders.resequenced?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

`@Component` replaces `@ApplicationScoped` — the route logic stays the same across both runtimes.

## Batch size, timeout, and why they matter for testing

The batch resequencer's two configuration knobs — `size` and `timeout` — directly affect how you design tests.

**Batch size** defines how many messages the resequencer collects before sorting and flushing.
In our example, the batch size is 10.
If you send exactly 3 messages in your test, the resequencer will *not* flush based on batch size alone — it will wait for the timeout.

**Timeout** is the safety net.
When the batch window expires (5000ms in our example), the resequencer flushes whatever it has collected, even if the batch is not full.
This means test execution time is bounded by this timeout — your test must be prepared to wait at least 5 seconds for output to appear.

This interaction between batch size and timeout is exactly the kind of behavior that unit tests with mocked endpoints cannot capture.
An integration test with real Kafka infrastructure surfaces the actual timing behavior and proves that the resequencer works end-to-end.

## Test infrastructure

Before diving into the test code, here is how the test infrastructure is set up.
Both runtimes use a shared pattern: a `BeforeSuite` action starts a Kafka broker via Testcontainers Docker Compose, and an `AfterSuite` action tears it down.

### Quarkus infrastructure setup

```java
@CitrusConfiguration
public class EipInfraSetup {

    @BindToRegistry
    public BeforeSuite beforeSuite() {
        return new TestDesigner().beforeSuite()
            .actions(
                testcontainers()
                    .compose()
                    .up()
                    .file("_infra/compose.yaml")
                    .waitFor("kafka", HttpWaitStrategy.newInstance()
                        .url("http://localhost:8090/")
                        .timeout(60000L))
            );
    }

    @BindToRegistry
    public AfterSuite afterSuite(CamelContext camelContext) {
        return new TestDesigner().afterSuite()
            .actions(
                camel().camelContext(camelContext).stop(),
                testcontainers()
                    .compose()
                    .down()
            );
    }
}
```

### Spring Boot infrastructure setup

```java
@Configuration
public class EipInfraSetup {

    @Bean
    public BeforeSuite beforeSuite() {
        return new TestDesigner().beforeSuite()
            .actions(
                testcontainers()
                    .compose()
                    .up()
                    .file("_infra/compose.yaml")
                    .waitFor("kafka", HttpWaitStrategy.newInstance()
                        .url("http://localhost:8090/")
                        .timeout(60000L))
            );
    }

    @Bean
    public AfterSuite afterSuite(CamelContext camelContext) {
        return new TestDesigner().afterSuite()
            .actions(
                camel().camelContext(camelContext).stop(),
                testcontainers()
                    .compose()
                    .down()
            );
    }
}
```

The difference is purely in the annotation style: `@CitrusConfiguration` with `@BindToRegistry` for Quarkus CDI, versus `@Configuration` with `@Bean` for Spring's application context.
The infrastructure lifecycle — start Kafka, wait for readiness, run tests, stop Camel, tear down containers — is identical.

## The resequencer test

The test scenario is straightforward: send three messages with out-of-order sequence numbers (3, 1, 2), then verify that the resequencer produces output on the resequenced topic.

### A shared test utility

Both runtimes implement the `EipTestSupport` interface, which provides a reusable `waitForCamelRouteStarted` method:

```java
public interface EipTestSupport extends TestActionSupport {

    default TestActionBuilder<?> waitForCamelRouteStarted(
            String routeId, CamelContext camelContext) {
        return repeatOnError()
                .until((i, context) -> i > 20)
                .autoSleep(Duration.ofSeconds(1))
                .actions(
                    camel().camelContext(camelContext)
                            .controlBus()
                            .route(routeId)
                            .status()
                            .result(ServiceStatus.Started),
                    sleep().seconds(5)
                );
    }
}
```

This utility uses Camel's Control Bus to poll the route status every second, up to 20 retries.
It ensures the `batch-resequencer` route is fully started and consuming from Kafka before the test sends any messages.
Without this guard, messages sent before the route is ready would be lost.

### Message template

All three messages use the same JSON template, with Citrus variable placeholders for dynamic values:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-00${id}",
  "item_sku": "SKU-TEST-${id}",
  "quantity": 1,
  "amount": ${amount},
  "destination_country": "${country}",
  "shipping_priority": "${priority}"
}
```

The template keeps the message body consistent while allowing each test run to use unique identifiers.

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
    class ResequencerTest {

        @Test
        public void shouldResequenceOutOfOrderMessages() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("amount", "50.00")
                    .variable("country", "US")
                    .variable("priority", "STANDARD")
            );

            t.given(waitForCamelRouteStarted("batch-resequencer", camelContext));

            t.given(
                print().message("Sending 3 messages with out-of-order sequence numbers: 3, 1, 2")
            );

            t.when(
                sequential()
                    .actions(
                        send()
                            .endpoint("kafka:eip.orders.sequenced")
                            .message()
                            .body(Resources.create("templates/order.json"))
                            .header("order-id", "${id}")
                            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
                            .header("sequenceNumber", 3),
                        send()
                            .endpoint("kafka:eip.orders.sequenced")
                            .message()
                            .body(Resources.create("templates/order.json"))
                            .header("order-id", "${id}")
                            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
                            .header("sequenceNumber", 1),
                        send()
                            .endpoint("kafka:eip.orders.sequenced")
                            .message()
                            .body(Resources.create("templates/order.json"))
                            .header("order-id", "${id}")
                            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
                            .header("sequenceNumber", 2)
                    )
            );

            t.then(
                repeatOnError()
                    .until((i, context) -> i > 10)
                    .autoSleep(Duration.ofSeconds(1))
                    .actions(
                        sequential()
                            .actions(
                                receive()
                                    .endpoint("kafka:eip.orders.resequenced?consumerGroup=citrus-resequenced-group")
                                    .message()
                                    .body(Resources.create("templates/order.json"))
                                    .header("sequenceNumber", 1),
                                receive()
                                    .endpoint("kafka:eip.orders.resequenced?consumerGroup=citrus-resequenced-group")
                                    .message()
                                    .body(Resources.create("templates/order.json"))
                                    .header("sequenceNumber", 2),
                                receive()
                                    .endpoint("kafka:eip.orders.resequenced?consumerGroup=citrus-resequenced-group")
                                    .message()
                                    .body(Resources.create("templates/order.json"))
                                    .header("sequenceNumber", 3)
                            )
                    )
            );
        }
    }
}
```

Let's walk through the test structure.

**Given — set up variables and wait for the route.**
The test generates a random 4-digit order ID and defines the order attributes.
It then waits for the `batch-resequencer` route to reach `Started` status via the Control Bus utility.

**When — send three out-of-order messages.**
Three messages go to the `eip.orders.sequenced` Kafka topic, each with the same order body but different `sequenceNumber` headers: 3, then 1, then 2.
All three share the same `order-id` and Kafka message key, simulating a single order's instructions arriving out of sequence.

**Then — verify resequenced output.**
The test receives the three output message from the `eip.orders.resequenced` topic and validates its body against the same template.
The `sequenceNumber` header validation ensures that the messages have been resequenced properly as expected.
The `repeatOnError()` block retries up to 10 times with 1-second intervals, giving the batch resequencer enough time to collect the messages, wait for its timeout window, sort the batch, and produce output.

### Spring Boot test

```java
@SpringBootTest(classes = AdvancedRoutingApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class ResequencerTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldResequenceOutOfOrderMessages() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("amount", "50.00")
                    .variable("country", "US")
                    .variable("priority", "STANDARD")
            );

            t.given(waitForCamelRouteStarted("batch-resequencer", camelContext));

            t.given(
                print().message("Sending 3 messages with out-of-order sequence numbers: 3, 1, 2")
            );

            t.when(
                sequential()
                    .actions(
                        send()
                            .endpoint("kafka:eip.orders.sequenced")
                            .message()
                            .body(Resources.create("templates/order.json"))
                            .header("order-id", "${id}")
                            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
                            .header("sequenceNumber", 3),
                        send()
                            .endpoint("kafka:eip.orders.sequenced")
                            .message()
                            .body(Resources.create("templates/order.json"))
                            .header("order-id", "${id}")
                            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
                            .header("sequenceNumber", 1),
                        send()
                            .endpoint("kafka:eip.orders.sequenced")
                            .message()
                            .body(Resources.create("templates/order.json"))
                            .header("order-id", "${id}")
                            .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
                            .header("sequenceNumber", 2)
                    )
            );

            t.then(
                repeatOnError()
                    .until((i, context) -> i > 10)
                    .autoSleep(Duration.ofSeconds(1))
                    .actions(
                        sequential()
                            .actions(
                                receive()
                                    .endpoint("kafka:eip.orders.resequenced?consumerGroup=citrus-resequenced-group")
                                    .message()
                                    .body(Resources.create("templates/order.json"))
                                    .header("sequenceNumber", 1),
                                receive()
                                    .endpoint("kafka:eip.orders.resequenced?consumerGroup=citrus-resequenced-group")
                                    .message()
                                    .body(Resources.create("templates/order.json"))
                                    .header("sequenceNumber", 2),
                                receive()
                                    .endpoint("kafka:eip.orders.resequenced?consumerGroup=citrus-resequenced-group")
                                    .message()
                                    .body(Resources.create("templates/order.json"))
                                    .header("sequenceNumber", 3)
                            )
                    )
            );
        }
    }
}
```

The test logic is identical.
The differences live entirely in the test class annotations and dependency injection:

| Concern                | Quarkus                                | Spring Boot                                |
|------------------------|----------------------------------------|--------------------------------------------|
| Test bootstrap         | `@QuarkusTest`                         | `@SpringBootTest` + `@CamelSpringBootTest` |
| Citrus integration     | `@CitrusSupport`                       | `@CitrusSpringSupport`                     |
| CamelContext injection | `@Inject` + `@BindToRegistry`          | `@Autowired`                               |
| TestCaseRunner scope   | Class-level field                      | Nested class field with `@CitrusResource`  |
| Infrastructure config  | `@CitrusConfiguration` auto-discovered | `@ContextConfiguration` explicit           |

## Why resequencer tests need patience

Unlike a content-based router or a wire tap, where the output appears almost immediately after the input, a batch resequencer deliberately introduces delay.
The resequencer holds messages until either the batch fills up or the timeout expires.

This has two implications for test design:

**1. The retry window must exceed the batch timeout.**
Our route uses a 5-second batch timeout.
The test's `repeatOnError()` retries up to 10 times with 1-second sleep intervals, giving the resequencer up to 10 seconds to produce output — comfortably longer than the 5-second timeout.
If the retry window were shorter than the batch timeout, the test would fail intermittently.

**2. Sending fewer messages than the batch size is intentional.**
The batch size is 10, but the test sends only 3 messages.
This forces the resequencer to rely on the timeout rather than the batch size to flush.
It is a deliberate test design choice: it verifies that the timeout mechanism works correctly, which is the more common production scenario (batches rarely fill to capacity under real workloads).

## Key takeaways

- **Batch resequencers are stateful and time-dependent.** Unlike stateless routing patterns, they accumulate messages before producing output. Integration tests must account for this delay.
- **`repeatOnError()` handles async verification gracefully.** Rather than inserting fixed sleeps, Citrus retries the receive action until it succeeds or the retry limit is reached — making tests both reliable and as fast as possible.
- **`waitForCamelRouteStarted` prevents message loss.** In a real Kafka environment, messages sent before the consumer route is ready are lost. The Control Bus check ensures the route is consuming before the test produces messages.
- **Three runtimes, one test pattern.** Whether you run on Quarkus, Spring Boot, or via Camel JBang with YAML DSL, the test structure — send out-of-order, wait for the batch window, verify resequenced output — stays the same. Only the bootstrap annotations and infrastructure wiring change.
- **Test the timeout path, not just the happy path.** Sending fewer messages than the batch size forces the timeout to trigger, exercising the code path that matters most in production.
