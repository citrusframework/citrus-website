---
layout: sample
title: Testing the Routing Slip Pattern with Citrus
name: routing-slip
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Routing Slip EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/routing-slip/
---

Content-based routers and message filters make a routing decision based on a fixed criteria: *this* message goes *there*.
What if the integration workflow requires sequences of decisions, and the sequence itself varies from message to message?

A standard domestic order passes through validation and carrier assignment.
An international order adds a customs classification step.
A hazardous-materials order adds a compliance check.
An international hazmat order needs both.

You could hard-code every combination into a massive content-based router, but every new order type would require code changes and the route logic would grow increasingly fragile.
The [Routing Slip](https://www.enterpriseintegrationpatterns.com/patterns/messaging/RoutingTable.html) pattern, described in *Enterprise Integration Patterns* by Hohpe and Woolf, offers a cleaner solution: attach the processing sequence to the message itself and let the router follow it.

Testing a routing slip raises a challenge that does not come up with simpler patterns.
With a content-based router, you verify that a message lands on the right destination.
With a routing slip, the same message must pass through *multiple* destinations in *sequence*, and the set of destinations varies depending on the message content.
The test must prove that the correct steps were visited — and that the wrong ones were skipped.

This is where [Citrus](https://citrusframework.org) comes in.
Citrus can send an order message to the input topic, then verify that the Camel routes corresponding to each slip step actually processed the message — proving the dynamic routing slip was assembled and executed correctly.
In this post, we walk through a complete example that tests a Routing Slip route with Citrus, running on both Quarkus and Spring Boot.

# The Camel route under test

The scenario is an order processing pipeline.
Orders arrive on a Kafka topic `eip.orders.placed` as JSON messages.
A processor inspects each order and builds a comma-separated list of `direct:` endpoint URIs — the routing slip.
The slip always starts with validation and ends with carrier assignment, but optional steps are inserted in between based on order properties.

Here is the Camel route on Quarkus:

```java
@ApplicationScoped
public class RoutingSlipRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=routing-slip-demo")
            .routeId("order-routing-slip")
            .unmarshal().json()
            .process(exchange -> {
                var body = exchange.getIn().getBody(Map.class);
                var steps = new ArrayList<String>();
                steps.add("direct:validate-order");

                boolean hazmat = Boolean.TRUE.equals(body.get("contains_hazmat"));
                if (hazmat) {
                    steps.add("direct:hazmat-compliance");
                }

                String country = (String) body.getOrDefault("destination_country", "US");
                if (!"US".equals(country)) {
                    steps.add("direct:customs-classification");
                }

                steps.add("direct:assign-carrier");
                exchange.getIn().setHeader("orderSlip", String.join(",", steps));
            })
            .log("Routing slip: ${header.orderSlip}")
            .routingSlip(header("orderSlip"), ",");

        from("direct:validate-order")
            .routeId("validate-order")
            .log("Validating order ${body[order_id]}");

        from("direct:hazmat-compliance")
            .routeId("hazmat-compliance")
            .log("HAZMAT compliance check for order ${body[order_id]}");

        from("direct:customs-classification")
            .routeId("customs-classification")
            .log("Customs classification for order ${body[order_id]} to ${body[destination_country]}");

        from("direct:assign-carrier")
            .routeId("assign-carrier")
            .log("Assigning carrier for order ${body[order_id]}");
    }
}
```

There are several things worth noting about this route.

First, the inline processor inspects the order body and builds the routing slip dynamically.
The `contains_hazmat` flag adds a `direct:hazmat-compliance` step, and a non-US `destination_country` adds `direct:customs-classification`.
The slip is stored in the `orderSlip` header as a comma-separated string like `direct:validate-order,direct:hazmat-compliance,direct:assign-carrier`.

Second, Camel's `routingSlip(header("orderSlip"), ",")` reads this header and routes the message through each endpoint in order.
The message visits every endpoint in the list, one after another.
Each step receives the same exchange — headers set by earlier steps are visible to later ones.

Third, each step in the slip is a standalone route with its own `routeId`.
This is important for testing: Camel's managed route beans expose exchange counters per route, so a test can verify which routes actually processed messages.

On Spring Boot, the route is identical except for the annotation:

```java
@Component
public class RoutingSlipRoute extends RouteBuilder {

    @Override
    public void configure() {
        // ... identical route logic
    }
}
```

The routing slip logic is runtime-agnostic.
Only the CDI annotation (`@ApplicationScoped` vs. `@Component`) changes between Quarkus and Spring Boot.

# What makes testing a routing slip different

Testing a routing slip differs from testing a content-based router in a fundamental way.
With a router, you send a message and verify it landed on the right output channel.
You have a clear input-output relationship: one message in, one message out on a specific topic.

With a routing slip, the message does not end up on a single output channel.
Instead, it passes *through* a series of internal processing steps.
The steps are `direct:` endpoints — they are in-process and do not produce output messages on external channels like Kafka topics.
The message enters on the input topic, flows through the slip, and the route completes.
There is no output message to consume and validate.

This means the test must verify the routing slip's behavior differently: not by consuming output messages, but by inspecting which Camel routes actually processed the exchange.
The key questions become:

- **Did the correct steps execute?** An international hazmat order should visit `validate-order`, `hazmat-compliance`, `customs-classification`, and `assign-carrier`.
- **Were the wrong steps skipped?** A standard domestic order should *not* visit `hazmat-compliance` or `customs-classification`.
- **Did any step fail?** A routing slip that silently drops a step due to an exception looks like a success from the outside.

Citrus addresses these challenges by leveraging Camel's JMX management API to inspect route-level exchange statistics.
Each route tracks how many exchanges it has completed and how many have failed.
By asserting these counters, the test can prove exactly which steps were visited.

# The message template

The test uses a single JSON template for the order, stored in `src/test/resources/templates/order.json`:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-${id}",
  "destination_country": "${country}",
  "contains_hazmat": ${hazmat}
}
```

The `${id}`, `${country}`, and `${hazmat}` placeholders are resolved by Citrus at runtime from test variables.
This single template serves all four test scenarios.
A domestic order sets `country` to `"US"` and `hazmat` to `false`.
An international hazmat order sets `country` to `"DE"` and `hazmat` to `true`.
The same template, different variables — the test data drives the routing slip behavior.

# Testing the four routing slip scenarios

The test suite covers four distinct routing paths, each as a separate nested test class.
Let us walk through them from simplest to most complex.

## Standard domestic order

The simplest case: a US domestic, non-hazmat order should only visit `validate-order` and `assign-carrier`, skipping both optional steps.

```java
@Nested
class RoutingSlipDomesticTest {

    @Test
    public void shouldRouteStandardDomesticOrder() {
        t.given(
            createVariables()
                .variable("id", "citrus:randomNumber(4)")
                .variable("country", "US")
                .variable("hazmat", false)
        );

        t.given(waitForCamelRouteStarted("order-routing-slip", camelContext));
        t.given(resetRouteStatistics(camelContext, "order-routing-slip",
                "validate-order", "assign-carrier",
                "hazmat-compliance", "customs-classification"));

        t.when(
            send()
                .endpoint("kafka:eip.orders.placed")
                .message()
                .body(Resources.create("templates/order.json"))
                .header("kafka.KEY", "${id}")
        );

        t.then(assertProcessedExchanges("order-routing-slip", 1, camelContext));
        t.then(assertProcessedExchanges("validate-order", 1, camelContext));
        t.then(assertProcessedExchanges("assign-carrier", 1, camelContext));
    }
}
```

The test follows Citrus's given-when-then structure.

The `given` phase creates test variables — a random order ID, `"US"` as the country, and `false` for hazmat.
It then waits for the `order-routing-slip` route to be fully started and connected to Kafka.
The `resetRouteStatistics` call zeroes out the exchange counters on all routes before the test runs — this ensures that counters from earlier tests in the suite do not pollute the assertions.

The `when` phase sends the order to the `eip.orders.placed` topic.
The message body is loaded from the `order.json` template, and Citrus resolves the placeholders with the variables defined above.

The `then` phase verifies that exactly the right routes processed exactly one exchange each.
The `assertProcessedExchanges` utility (described below) uses Camel's managed route beans to check the exchange counter, confirming the routing slip ran to completion.
Because the counters were reset before the test, the assertion can use an exact count of `1` instead of `>= 1` — a stronger guarantee that the test is observing only its own exchange.

## Hazmat order

A US order containing hazardous materials should add the `hazmat-compliance` step to the slip:

```java
@Nested
class RoutingSlipHazmatTest {

    @Test
    public void shouldRouteHazmatOrder() {
        t.given(
            createVariables()
                .variable("id", "citrus:randomNumber(4)")
                .variable("country", "US")
                .variable("hazmat", true)
        );

        t.given(waitForCamelRouteStarted("order-routing-slip", camelContext));
        t.given(resetRouteStatistics(camelContext, "order-routing-slip",
                "validate-order", "assign-carrier",
                "hazmat-compliance", "customs-classification"));

        t.when(
            send()
                .endpoint("kafka:eip.orders.placed")
                .message()
                .body(Resources.create("templates/order.json"))
                .header("kafka.KEY", "${id}")
        );

        t.then(assertProcessedExchanges("validate-order", 1, camelContext));
        t.then(assertProcessedExchanges("hazmat-compliance", 1, camelContext));
        t.then(assertProcessedExchanges("assign-carrier", 1, camelContext));
    }
}
```

The only variable change is `hazmat` set to `true`.
The assertions now verify the complete expected slip: `validate-order`, `hazmat-compliance`, and `assign-carrier` — each with exactly one exchange.

## International order

A non-US order without hazmat should include `customs-classification` but skip `hazmat-compliance`:

```java
@Nested
class RoutingSlipInternationalTest {

    @Test
    public void shouldRouteInternationalOrder() {
        t.given(
            createVariables()
                .variable("id", "citrus:randomNumber(4)")
                .variable("country", "DE")
                .variable("hazmat", false)
        );

        t.given(waitForCamelRouteStarted("order-routing-slip", camelContext));
        t.given(resetRouteStatistics(camelContext, "order-routing-slip",
                "validate-order", "assign-carrier",
                "hazmat-compliance", "customs-classification"));

        t.when(
            send()
                .endpoint("kafka:eip.orders.placed")
                .message()
                .body(Resources.create("templates/order.json"))
                .header("kafka.KEY", "${id}")
        );

        t.then(assertProcessedExchanges("validate-order", 1, camelContext));
        t.then(assertProcessedExchanges("customs-classification", 1, camelContext));
        t.then(assertProcessedExchanges("assign-carrier", 1, camelContext));
    }
}
```

Setting `country` to `"DE"` triggers the customs classification step.
The assertions verify the complete expected slip: `validate-order`, `customs-classification`, and `assign-carrier`.

## International hazmat order — the full pipeline

The most comprehensive test sends an order that triggers *every* optional step and then verifies *all* routes in the slip:

```java
@Nested
class RoutingSlipFullPipelineTest {

    @Test
    public void shouldRouteInternationalHazmatOrderThroughAllSteps() {
        t.given(
            createVariables()
                .variable("id", "citrus:randomNumber(4)")
                .variable("country", "DE")
                .variable("hazmat", true)
        );

        t.given(waitForCamelRouteStarted("order-routing-slip", camelContext));
        t.given(resetRouteStatistics(camelContext, "order-routing-slip",
                "validate-order", "assign-carrier",
                "hazmat-compliance", "customs-classification"));

        t.when(
            send()
                .endpoint("kafka:eip.orders.placed")
                .message()
                .body(Resources.create("templates/order.json"))
                .header("kafka.KEY", "${id}")
        );

        t.then(assertProcessedExchanges("order-routing-slip", 1, camelContext));
        t.then(assertProcessedExchanges("validate-order", 1, camelContext));
        t.then(assertProcessedExchanges("hazmat-compliance", 1, camelContext));
        t.then(assertProcessedExchanges("customs-classification", 1, camelContext));
        t.then(assertProcessedExchanges("assign-carrier", 1, camelContext));
    }
}
```

This test sets both `country` to `"DE"` and `hazmat` to `true`, producing the maximum-length routing slip: `direct:validate-order,direct:hazmat-compliance,direct:customs-classification,direct:assign-carrier`.

The `then` phase asserts against *every* route in the slip.
Each `assertProcessedExchanges` call verifies that the named route completed exactly one exchange without errors.
If any step was skipped or failed, the corresponding assertion catches it.

# Verifying route execution with Camel's management API

The `EipTestSupport` interface provides two core utilities for routing slip tests: resetting route statistics before each test, and asserting the exchange counts afterward.

## Resetting route statistics

When multiple tests run in the same suite, exchange counters accumulate across tests.
If the domestic test increments `validate-order` to 1, the hazmat test would see it at 2 — making exact-count assertions impossible without a reset.

The `resetRouteStatistics` utility zeroes out the counters for a set of routes before each test:

```java
default TestActionBuilder<?> resetRouteStatistics(
        CamelContext camelContext, String... routeIds) {
    return sequential()
            .actions(Arrays.stream(routeIds)
                    .map(routeId -> resetRouteStatistics(routeId, camelContext))
                    .collect(Collectors.toSet())
                    .toArray(TestActionBuilder[]::new));
}

default TestActionBuilder<?> resetRouteStatistics(
        String routeId, CamelContext camelContext) {
    return () -> (context) -> {
        ManagedCamelContext managedContext = camelContext
                .getCamelContextExtension()
                .getContextPlugin(ManagedCamelContext.class);
        ManagedRouteMBean routeMBean =
                managedContext.getManagedRoute(routeId);

        if (routeMBean != null) {
            routeMBean.reset(true);
        } else {
            throw new CitrusRuntimeException(
                "No managed route for routeId '%s'"
                    .formatted(routeId));
        }
    };
}
```

The `reset(true)` call performs a deep reset, cascading to all processors within the route.
Each test resets *all* routes — not just the ones it expects to hit — so that a route that should *not* be visited stays at zero.

## Asserting exchange counts

The `assertProcessedExchanges` utility inspects Camel's internal route statistics to verify which steps the message visited.
Unlike pattern tests that consume output messages, this approach works with `direct:` endpoints that produce no external output:

```java
default TestActionBuilder<?> assertProcessedExchanges(
        String routeId, long expected, CamelContext camelContext) {
    return assertProcessedExchanges(routeId, it -> it == expected, camelContext);
}

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
```

The utility uses Camel's `ManagedCamelContext` to access the `ManagedRouteMBean` for a given route ID.
Each managed route bean tracks two key counters: completed exchanges and failed exchanges.

The method first checks for failures — if any exchange failed on the route, the test fails immediately with a clear error message.
Then it checks the completed count against the provided predicate.
The convenience overload `assertProcessedExchanges(routeId, 1, camelContext)` asserts an exact count — the most common case when route statistics have been reset before each test.
The `repeatOnError` wrapper retries up to 20 times with one-second pauses, accommodating the asynchronous nature of Kafka-based processing where there is a delay between sending a message and the route completing its execution.

This approach has an important advantage over output-message verification: it directly proves which processing steps the message visited, not just where it ended up.
A routing slip with four steps produces no output messages if the steps are all `direct:` endpoints — there is nothing to consume.
But each step's route counter increments, providing a precise record of the processing path.

# Waiting for the Camel route

Camel routes that consume from Kafka take a moment to start and subscribe to the broker.
If the test sends a message before the route's consumer is ready, the message sits unprocessed and the test times out.

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
Every test class implements `EipTestSupport` and calls `waitForCamelRouteStarted(...)` in its `given` phase.

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
    class RoutingSlipDomesticTest {
        // ... test methods
    }
}
```

On Spring Boot, the test uses `@CitrusSpringSupport` alongside Spring Boot and Camel test annotations:

```java
@SpringBootTest(classes = ComposedRoutingApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class RoutingSlipDomesticTest {

        @CitrusResource
        TestCaseRunner t;

        // ... test methods
    }
}
```

On Spring Boot, the `TestCaseRunner` is declared in the `@Nested` inner class, while on Quarkus it lives at the outer class level — a structural difference driven by how each test framework manages injection scopes.
The actual test methods inside each nested class are identical on both runtimes.

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
cd examples/10-composed-routing/quarkus
mvn verify

# Spring Boot
cd examples/10-composed-routing/spring-boot
mvn verify
```

The test suite starts the Kafka broker via Testcontainers, boots the Camel application, runs all four routing slip tests, and tears everything down.

# Key takeaways

Testing a Routing Slip means verifying that a message visits the correct sequence of processing steps — and skips the ones it should not visit.
Unlike single-destination patterns where you check output messages, a routing slip test must prove which internal routes were activated by the dynamic slip.

Here is what Citrus brings to this problem:

- **Reset-and-assert pattern** — The `resetRouteStatistics` utility zeroes out exchange counters before each test, enabling exact-count assertions. Combined with `assertProcessedExchanges`, this proves not only which steps were visited but that each step processed exactly the expected number of exchanges — a stronger guarantee than `>= 1` checks.
- **Route-level exchange assertions** — The `assertProcessedExchanges` utility uses Camel's `ManagedRouteMBean` to inspect completed and failed exchange counters per route. This directly proves which steps were visited without relying on output messages.
- **Test variables for data-driven scenarios** — A single `order.json` template serves all four test scenarios. The `${country}` and `${hazmat}` variables drive the routing slip behavior, keeping the tests concise and the intent clear.
- **Retry-tolerant verification** — The `repeatOnError` wrapper accommodates Kafka's asynchronous processing model, retrying assertions until the route counters reflect the completed exchange.
- **Multi-runtime portability** — The same test logic runs on Quarkus and Spring Boot. Only the wiring annotations change; the send, wait, and assertion logic stays identical.

The complete source code for this example is available on GitHub:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/10-composed-routing/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/10-composed-routing/spring-boot)

Give it a try, and let us know what you think!
