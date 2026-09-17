---
layout: sample
title: Testing the Load Balancer Pattern with Citrus
name: load-balancer
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Load Balancer EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/load-balancer/
---

Most routing patterns deal with *where* a message should go based on its content or some predetermined sequence of steps.
The Load Balancer pattern addresses a different concern: when multiple equivalent processors can handle a message, *which one should get it?*

Consider an order fulfillment system with three regional fulfillment centers — East, Central, and West.
Each center is capable of processing any order, but no single center should handle all the traffic.
You need to distribute orders evenly across the three centers to avoid overloading one while the others sit idle.

In a pure Kafka architecture, consumer group partition assignment handles load distribution automatically.
But not every downstream endpoint is Kafka.
When the targets are HTTP services, in-process handlers, or `direct:` routes within the same application, you need explicit load balancing at the routing level.

The [Load Balancer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageDispatcher.html) pattern, rooted in the messaging patterns described by Hohpe and Woolf, distributes messages across a set of processing endpoints using a strategy — round-robin, random, sticky, failover, or custom.
[Apache Camel](https://camel.apache.org) implements this with the `loadBalance()` EIP, which wraps multiple `to()` endpoints and selects one per message according to the chosen strategy.

Testing a load balancer is unlike testing a content-based router or a message filter.
With those patterns, a single message goes to a single deterministic destination.
With a load balancer, you need to send *multiple* messages and then verify that *each* downstream processor received at least some of them.
The test must prove distribution happened — that all three fulfillment centers participated, not just one.

This is where [Citrus](https://citrusframework.org) comes in.
Citrus can send a batch of messages to the load-balanced input topic and then verify, via Camel's route management API, that every downstream route processed at least one exchange.
In this post, we walk through a complete example that tests a round-robin Load Balancer with Citrus, running on both Quarkus and Spring Boot.

# The Camel route under test

The scenario is a fulfillment center dispatcher.
Orders arrive on a Kafka topic `eip.orders.loadbalanced` as JSON messages.
The load balancer distributes each order to one of three regional fulfillment centers using a round-robin strategy.
Each fulfillment center is a `direct:` route that processes the order by stamping it with the center's name and warehouse code.

Here is the Camel route on Quarkus:

```java
@ApplicationScoped
public class LoadBalancerRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.loadbalanced?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=loadbalancer-demo")
            .routeId("load-balancer-demo")
            .unmarshal().json()
            .log("Load Balancer received order ${body[order_id]}")
            .loadBalance().roundRobin()
                .to("direct:fulfillment-center-east",
                    "direct:fulfillment-center-central",
                    "direct:fulfillment-center-west")
            .end();

        from("direct:fulfillment-center-east")
            .routeId("fulfillment-center-east")
            .log("EAST fulfillment center processing order ${body[order_id]}")
            .process(exchange -> {
                var body = exchange.getIn().getBody(java.util.Map.class);
                body.put("fulfillment_center", "EAST");
                body.put("warehouse_code", "WH-NYC-01");
            })
            .log("Order ${body[order_id]} assigned to EAST (WH-NYC-01)");

        from("direct:fulfillment-center-central")
            .routeId("fulfillment-center-central")
            .log("CENTRAL fulfillment center processing order ${body[order_id]}")
            .process(exchange -> {
                var body = exchange.getIn().getBody(java.util.Map.class);
                body.put("fulfillment_center", "CENTRAL");
                body.put("warehouse_code", "WH-CHI-01");
            })
            .log("Order ${body[order_id]} assigned to CENTRAL (WH-CHI-01)");

        from("direct:fulfillment-center-west")
            .routeId("fulfillment-center-west")
            .log("WEST fulfillment center processing order ${body[order_id]}")
            .process(exchange -> {
                var body = exchange.getIn().getBody(java.util.Map.class);
                body.put("fulfillment_center", "WEST");
                body.put("warehouse_code", "WH-LAX-01");
            })
            .log("Order ${body[order_id]} assigned to WEST (WH-LAX-01)");
    }
}
```

There are several things worth noting about this route.

First, the `loadBalance().roundRobin()` call wraps three `direct:` endpoints.
Camel cycles through them in order: the first message goes to East, the second to Central, the third to West, and then the cycle repeats.
Round-robin is the simplest load balancing strategy — it guarantees perfectly even distribution when all endpoints are healthy.

Second, each fulfillment center is a standalone route with its own `routeId`.
The East center has route ID `fulfillment-center-east`, Central has `fulfillment-center-central`, and West has `fulfillment-center-west`.
This is important for testing: Camel's JMX management beans track exchange counters per route, so the test can verify that each center actually received and processed messages.

Third, each center stamps the order body with a `fulfillment_center` name and `warehouse_code`.
In a real system, this step might call an external warehouse API or update a database.
For testing purposes, the in-memory processing gives us a clear signal that the message reached the intended destination.

On Spring Boot, the route is identical except for the annotation:

```java
@Component
public class LoadBalancerRoute extends RouteBuilder {

    @Override
    public void configure() {
        // ... identical route logic
    }
}
```

# How round-robin load balancing works

Camel's round-robin strategy maintains an internal counter that cycles through the list of endpoints.
The counter increments with each message, and the endpoint is selected by `counter % numberOfEndpoints`.
For three endpoints, the pattern is:

| Message # | Endpoint selected                   |
|-----------|-------------------------------------|
| 1         | `direct:fulfillment-center-east`    |
| 2         | `direct:fulfillment-center-central` |
| 3         | `direct:fulfillment-center-west`    |
| 4         | `direct:fulfillment-center-east`    |
| 5         | `direct:fulfillment-center-central` |
| ...       | ...                                 |

This means that sending exactly three messages guarantees each endpoint receives exactly one.
That is a useful property for testing — three messages create a deterministic, fully covered distribution.

Camel also supports other strategies that suit different requirements:

- **Failover** — tries the first endpoint; if it fails, tries the next. Useful for redundant HTTP services.
- **Sticky** — routes by a hash of a key (like customer ID) so the same customer always hits the same instance. Useful for stateful processors.
- **Random** — selects an endpoint at random. Less predictable for testing, but ensures uniform distribution at scale.
- **Custom** — implements a custom selection strategy, for example based on endpoint health or current load.

The round-robin strategy is the easiest to test because of its deterministic assignment, so this example focuses on it.

# What makes testing a load balancer different

Testing a load balancer differs from testing other routing patterns in two fundamental ways.

First, **a single message is not enough**.
With a content-based router, one message reveals the routing decision — the message either goes to topic A or topic B.
With a load balancer, a single message tells you only that *one* endpoint received it.
You cannot prove distribution happened from a single observation.
The test must send *multiple* messages — at minimum, as many as there are downstream endpoints.

Second, **the verification target is aggregate, not individual**.
The test does not need to prove that message 1 went to East and message 2 went to Central — that would couple the test to the internal counter state, which may differ across runs.
Instead, the test proves that *all three fulfillment centers participated* — each received at least one exchange.
This is a distribution assertion: not "exactly this message went exactly here," but "all destinations received work."

This shifts the verification approach from message-level validation (consuming a specific output message) to route-level statistics (checking exchange counters across multiple routes).
Citrus handles this by leveraging Camel's `ManagedRouteMBean` API, which exposes per-route completed and failed exchange counts.

# The message template

The test uses a single JSON template for orders, stored in `src/test/resources/templates/order.json`:

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

The `${id}`, `${amount}`, `${country}`, and `${priority}` placeholders are resolved by Citrus at runtime from test variables.
The template is the same one used by other pattern tests in the example suite.
For the load balancer test, the order content does not matter — round-robin routing ignores the message body entirely.
What matters is that *three* distinct messages are sent, each with a fresh ID.

# The load balancer test

The test sends three orders and verifies that all three fulfillment centers received at least one.
Here is the test on Quarkus:

```java
@Nested
class LoadBalancerTest {

    @Test
    public void shouldDistributeOrdersAcrossFulfillmentCenters() {
        t.given(
            createVariables()
                .variable("amount", "75.00")
                .variable("country", "US")
                .variable("priority", "STANDARD")
        );

        t.given(waitForCamelRouteStarted("load-balancer-demo", camelContext));

        t.when(
            iterate()
                .times(3)
                .actions(
                    createVariables().variable("id", "citrus:randomNumber(4)"),
                    send()
                        .endpoint("kafka:eip.orders.loadbalanced")
                        .message()
                        .body(Resources.create("templates/order.json"))
                        .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
                )
        );

        t.then(assertProcessedExchanges("load-balancer-demo", it -> it >= 3, camelContext));
        t.then(assertProcessedExchanges("fulfillment-center-east", it -> it >= 1, camelContext));
        t.then(assertProcessedExchanges("fulfillment-center-central", it -> it >= 1, camelContext));
        t.then(assertProcessedExchanges("fulfillment-center-west", it -> it >= 1, camelContext));
    }
}
```

Let us walk through each phase.

## The given phase

The `given` phase sets up shared test variables for the order template — `amount`, `country`, and `priority`.
These values are the same for all three messages because the load balancer's round-robin strategy is content-agnostic.
The route then waits for `load-balancer-demo` to be fully started and connected to Kafka.

## The when phase — sending three messages

The `when` phase uses a loop to send three orders.
Each iteration generates a fresh random `id` via `citrus:randomNumber(4)` and sends the order to `eip.orders.loadbalanced`.
The `KafkaMessageHeaders.MESSAGE_KEY` is set to `${id}` for consistent partitioning.

The `iterate` loop is important: three messages match the three downstream endpoints in the round-robin cycle.
Sending fewer than three would not guarantee full coverage.
Sending exactly three, with round-robin, guarantees that each fulfillment center receives exactly one.

## The then phase — verifying distribution

The `then` phase is where the load-balancer-specific verification happens.
It consists of four `assertProcessedExchanges` calls:

1. `load-balancer-demo` must have processed at least 3 exchanges — confirming all three messages were consumed from Kafka and routed.
2. `fulfillment-center-east` must have processed at least 1 exchange.
3. `fulfillment-center-central` must have processed at least 1 exchange.
4. `fulfillment-center-west` must have processed at least 1 exchange.

The use of `it -> it >= 1` rather than `it -> it == 1` is deliberate.
While round-robin guarantees exactly one message per center when sending three, previous test runs in the same JVM (other test classes also send to the load-balanced topic) may have already incremented the counters.
Using `>= 1` makes the assertion resilient to execution order within the test suite.

If any of the four assertions fail — for example, if a bug in the route configuration accidentally sent all three messages to East — the test catches it immediately.

# Verifying route execution with Camel's management API

The `assertProcessedExchanges` utility is the core verification mechanism for the load balancer test.
It inspects Camel's internal route statistics through the JMX management API:

```java
public interface EipTestSupport extends TestActionSupport {

    default TestActionBuilder<?> assertProcessedExchanges(
            String routeId, Predicate<Long> check, CamelContext camelContext) {
        return repeatOnError()
                .until((i, context) -> i > 20)
                .autoSleep(Duration.ofSeconds(1))
                .actions(
                    context -> {
                        ManagedCamelContext managedContext = camelContext
                                .getCamelContextExtension()
                                .getContextPlugin(ManagedCamelContext.class);
                        ManagedRouteMBean routeMBean =
                                managedContext.getManagedRoute(routeId);

                        if (routeMBean != null) {
                            long failed = routeMBean.getExchangesFailed();
                            if (failed > 0) {
                                throw new ValidationException(
                                    "Route '%s' has %d failed exchanges"
                                        .formatted(routeId, failed));
                            }
                            long completed = routeMBean.getExchangesCompleted();
                            if (!check.test(completed)) {
                                throw new ValidationException(
                                    "Route '%s' has %d completed exchanges"
                                        .formatted(routeId, completed));
                            }
                        } else {
                            throw new CitrusRuntimeException(
                                "No managed route for routeId '%s'"
                                    .formatted(routeId));
                        }
                    }
                );
    }
}
```

The utility retrieves the `ManagedRouteMBean` for the given route ID and checks two counters.
First, it verifies that no exchanges failed — a load balancer that silently drops a message to a failing endpoint should not pass the test.
Then it verifies the completed count against the provided predicate.

The `repeatOnError` wrapper retries up to 20 times with one-second pauses.
This accommodates the asynchronous nature of Kafka-based processing: after sending three messages, there is a delay before all three have been consumed, routed through the load balancer, and fully processed by the downstream centers.
The retry loop ensures the test waits for processing to complete rather than asserting against stale counters.

This approach is particularly well suited for load balancer testing.
The load balancer distributes to `direct:` endpoints — in-process routes that produce no output messages on external channels.
There is nothing to consume from a Kafka output topic.
Instead, the route counters serve as proof that the distribution happened correctly.

# Waiting for the Camel route

Camel routes that consume from Kafka take a moment to start and subscribe to the broker.
If the test sends messages before the route's consumer is ready, the messages sit unprocessed and the assertions time out.

The example addresses this with a reusable utility that uses Camel's Control Bus to poll the route status:

```java
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
```

This polls the Camel context up to 20 times with one-second pauses.
Once the route reports `Started`, it waits an additional five seconds for the Kafka consumer to fully connect and receive its partition assignments.
The test calls `waitForCamelRouteStarted("load-balancer-demo", camelContext)` in its `given` phase before sending any messages.

# Setting up the test infrastructure

The route consumes from Kafka, so the test needs a running broker.
Citrus manages this lifecycle with Testcontainers — a Docker Compose file defines a single-node Kafka (KRaft mode) setup along with a Kafka UI for observability.

On Quarkus, the infrastructure setup uses `@CitrusConfiguration` with `@BindToRegistry`:

```java
@CitrusConfiguration
public class EipInfraSetup implements TestActionSupport {

    @BindToRegistry
    public BeforeSuite startInfra() {
        return beforeSuite().actions(
                    testcontainers().compose()
                            .up("_infra/compose.yaml")
                            .containerName("eip-infra")
                            .autoRemove(false),
                    waitFor()
                            .http()
                            .url("http://localhost:8090")
                            .seconds(25)
                ).build();
    }

    @BindToRegistry
    public AfterSuite stopInfra() {
        return afterSuite().actions(
                    camel().camelContext().stop(),
                    testcontainers().compose()
                            .down()
                            .containerName("eip-infra")
                ).build();
    }
}
```

On Spring Boot, the same logic uses `@Configuration` and `@Bean`:

```java
@Configuration
public class EipInfraSetup implements TestActionSupport {

    @Bean
    public BeforeSuite startInfra() {
        return beforeSuite().actions(
                    testcontainers().compose()
                            .up("_infra/compose.yaml")
                            .containerName("eip-infra")
                            .autoRemove(false),
                    waitFor()
                            .http()
                            .url("http://localhost:8090")
                            .seconds(25)
                ).build();
    }

    @Bean
    public AfterSuite stopInfra() {
        return afterSuite().actions(
                    camel().camelContext().stop(),
                    testcontainers().compose()
                            .down()
                            .containerName("eip-infra")
                ).build();
    }
}
```

The test actions are identical — only the bean registration mechanism differs between the two runtimes.

# Running on Quarkus and Spring Boot

The Camel route logic is identical across both runtimes.
The test code differences are limited to the wiring annotations.

On Quarkus, the test class uses `@CitrusSupport` alongside `@QuarkusTest`, and the `CamelContext` is injected with CDI:

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
    class LoadBalancerTest {
        // ... test methods
    }
}
```

On Spring Boot, the test uses `@CitrusSpringSupport` alongside Spring Boot and Camel test annotations:

```java
@SpringBootTest(classes = AdvancedRoutingApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class LoadBalancerTest {

        @CitrusResource
        TestCaseRunner t;

        // ... test methods
    }
}
```

On Spring Boot, the `TestCaseRunner` is declared in the `@Nested` inner class, while on Quarkus it lives at the outer class level — a structural difference driven by how each test framework manages injection scopes.
The actual test methods inside `LoadBalancerTest` are identical on both runtimes.

# Test dependencies

Both runtime variants share the same core Citrus dependencies.
The runtime adapter is the only difference — `citrus-quarkus` for Quarkus, `citrus-spring` for Spring Boot:

```xml
<!-- Quarkus runtime adapter -->
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>

<!-- OR Spring Boot runtime adapter -->
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-spring</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Both runtimes share these common dependencies:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-camel</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-kafka</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-testcontainers</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

The `citrus-camel` module provides the Control Bus integration used in `waitForCamelRouteStarted` and the managed route bean access in `assertProcessedExchanges`.
The `citrus-kafka` module provides the Kafka send actions.
The `citrus-testcontainers` module manages the Docker Compose lifecycle for the Kafka broker.

# Running the tests

Make sure Docker (or Podman) is running, then execute:

```bash
# Quarkus
cd examples/11-advanced-routing/quarkus
mvn verify

# Spring Boot
cd examples/11-advanced-routing/spring-boot
mvn verify
```

The test suite starts the Kafka broker via Testcontainers, boots the Camel application, runs the load balancer test (along with the other pattern tests in the suite), and tears everything down.

# Key takeaways

Testing a Load Balancer means verifying that messages are distributed across all downstream processors — not just routed to one.
Unlike content-based routing where a single message reveals the routing decision, a load balancer test requires multiple messages and aggregate verification across all destinations.

Here is what Citrus brings to this problem:

- **Multi-message send with a loop** — A `for` loop sends three messages, one for each downstream endpoint in the round-robin cycle. Each message gets a fresh `id` via `citrus:randomNumber(4)`, ensuring unique order identities.
- **Route-level exchange assertions** — Four `assertProcessedExchanges` calls verify the distribution: the load balancer route processed all three messages, and each fulfillment center received at least one. This proves the round-robin strategy is working.
- **Failure detection** — The `assertProcessedExchanges` utility checks for failed exchanges before checking completed counts. If a downstream center threw an exception, the test fails immediately with a clear error — not a silent miscount.
- **Retry-tolerant verification** — The `repeatOnError` wrapper accommodates the delay between sending Kafka messages and the route completing its processing, retrying assertions until the counters stabilize.
- **Multi-runtime portability** — The same test logic runs on Quarkus and Spring Boot. Only the wiring annotations change; the send loop, wait, and assertion logic stays identical.

The complete source code for this example is available on GitHub:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/11-advanced-routing/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/11-advanced-routing/spring-boot)

Give it a try, and let us know what you think!
