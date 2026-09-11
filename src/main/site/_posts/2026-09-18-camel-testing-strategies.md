---
layout: post
title: Camel Testing Strategies - From Unit to Integration Tests
short-title: Camel Testing Strategies
author: Christoph Deppisch
github: christophd
categories: [blog]
---

Every [Apache Camel](https://camel.apache.org) project needs tests, but which kind? A unit test with `AdviceWith` and `MockEndpoint` runs in milliseconds and tells you whether the filter predicate is correct. But it never touches a Kafka broker, never serializes a message to JSON bytes, never deals with consumer group offsets. An integration test with [Citrus](https://citrusframework.org) starts real infrastructure, sends through real transports, and catches the bugs that in-memory mocking cannot see — but it takes seconds, not milliseconds, and requires Docker.

The answer is not one or the other. A well-tested Camel project uses both. Unit tests provide fast feedback on routing logic during development. Integration tests provide confidence that the full pipeline works end-to-end before deployment. They test different things, and the bugs they catch barely overlap.

This post walks through both approaches side by side, using the same routes as examples. We start with Camel's built-in unit testing tools, then show how Citrus integration tests cover the gaps that have been missed. Along the way, we build a clear guidance for deciding which tool to use when.

# Unit testing with AdviceWith and MockEndpoint

Camel's built-in test support lets you replace a route's real endpoints with in-memory stubs. The key tools are `AdviceWith` — which modifies a route's definition before it starts — and `MockEndpoint` — which captures messages for assertion.

## A route designed for both testing levels

Here is an `OrderFilterRoute` that filters high-value orders. Notice the design: the Kafka consumer delegates to a `direct:filter-order` endpoint, and the filter's output goes to a `direct:high-value-orders` endpoint:

```java
@ApplicationScoped
public class OrderFilterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("direct:filter-order")
            .routeId("order-filter")
            .unmarshal().json(java.util.Map.class)
            .filter().simple("${body[amount]} >= 100")
                .log("High-value order ${body[order_id]}: $${body[amount]}")
                .to("direct:high-value-orders")
            .end();

        from("direct:high-value-orders")
            .routeId("high-value-handler")
            .log("High-value order received for priority processing");

        from("kafka:eip.orders.placed?brokers={{kafka.brokers}}&groupId=filter-service")
            .routeId("kafka-order-filter")
            .to("direct:filter-order");
    }
}
```

This layered design is intentional. The core logic lives in `direct:filter-order`, which can be tested without Kafka. The Kafka consumer is a thin adapter that feeds into the same `direct:` endpoint. Unit tests target the logic layer; integration tests target the full Kafka path.

## The unit test

```java
@QuarkusTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class OrderFilterRouteTest {

    @Inject
    CamelContext camelContext;

    @Inject
    ProducerTemplate producer;

    @BeforeAll
    void adviceRoutes() throws Exception {
        AdviceWith.adviceWith(camelContext, "kafka-order-filter", route -> {
            route.replaceFromWith("direct:test-kafka-input");
        });
        AdviceWith.adviceWith(camelContext, "high-value-handler", route -> {
            route.weaveAddLast().to("mock:high-value");
        });
    }

    @BeforeEach
    void resetMocks() {
        MockEndpoint.resetMocks(camelContext);
    }

    @Test
    void highValueOrderPassesFilter() throws Exception {
        MockEndpoint mock = camelContext.getEndpoint("mock:high-value", MockEndpoint.class);
        mock.expectedMessageCount(1);

        String order = """
            {"order_id": 2001, "amount": 250.00, "customer_id": "C-100"}
            """;
        producer.sendBody("direct:filter-order", order);

        mock.assertIsSatisfied();
    }

    @Test
    void lowValueOrderIsFiltered() throws Exception {
        MockEndpoint mock = camelContext.getEndpoint("mock:high-value", MockEndpoint.class);
        mock.expectedMessageCount(0);

        String order = """
            {"order_id": 2002, "amount": 49.99, "customer_id": "C-101"}
            """;
        producer.sendBody("direct:filter-order", order);

        mock.assertIsSatisfied();
    }

    @Test
    void borderlineOrderPassesFilter() throws Exception {
        MockEndpoint mock = camelContext.getEndpoint("mock:high-value", MockEndpoint.class);
        mock.expectedMessageCount(1);

        String order = """
            {"order_id": 2003, "amount": 100.00, "customer_id": "C-102"}
            """;
        producer.sendBody("direct:filter-order", order);

        mock.assertIsSatisfied();
    }
}
```

The `@BeforeAll` method does two things. First, it replaces the Kafka consumer route's `from:` with a `direct:` endpoint, so no Kafka broker is needed. Second, it appends a `mock:high-value` endpoint to the `high-value-handler` route, capturing any message that passes the filter.

Each test sends a JSON string to `direct:filter-order` via `ProducerTemplate` and checks whether `mock:high-value` received the expected number of messages. The boundary test with `amount: 100.00` catches off-by-one errors in the `>= 100` predicate.

## What unit tests catch

This approach is excellent for testing **routing logic in isolation**:

- Filter predicates — does the `simple("${body[amount]} >= 100")` expression evaluate correctly?
- Content-Based Router conditions — does each `when()` branch fire for the right input?
- Bean method results — does the enrichment service return the expected data?
- Boundary conditions — what happens at exactly 100.00?

The tests run fast because they operate entirely in memory. No Docker, no network calls, no serialization overhead.

## What unit tests miss

The test sends a JSON *string* and receives a Java *object*. It never tests whether the route correctly marshals the output back to JSON before sending to Kafka. A route that does `unmarshal().json()` → filter → `to("kafka:...")` without a `marshal().json()` call would produce `{order_id=2001, amount=250.0, customer_id=C-100}` on the Kafka topic — Java `Map.toString()` output, not valid JSON. The unit test would never notice because `MockEndpoint` captures the in-memory Java object, not the serialized bytes.

Unit tests also cannot verify:

- **Kafka consumer group behavior** — offset management, rebalancing, partition assignment.
- **Header propagation** — whether Kafka message keys and custom headers survive the transport.
- **Multi-hop async flows** — where a message passes through multiple Kafka topics and routes.
- **Infrastructure configuration** — broker connection strings, topic auto-creation, timeouts.

# Testing routes with multiple branches

The same `AdviceWith` pattern scales to Content-Based Routers. Here is a route that classifies orders by destination:

```java
@ApplicationScoped
public class OrderValidationRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("direct:validate-order")
            .routeId("order-validation")
            .unmarshal().json(java.util.Map.class)
            .choice()
                .when().simple("${body[shipping_type]} == 'HAZMAT'")
                    .to("direct:hazmat")
                .when().simple("${body[country]} != 'US'")
                    .to("direct:international")
                .otherwise()
                    .to("direct:domestic")
            .end();
    }
}
```

The unit test adds mock endpoints to all three branches and verifies each one independently:

```java
@QuarkusTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class OrderValidationRouteTest {

    @Inject
    CamelContext camelContext;

    @Inject
    ProducerTemplate producer;

    @BeforeAll
    void adviceRoutes() throws Exception {
        AdviceWith.adviceWith(camelContext, "domestic-handler", route -> {
            route.weaveAddLast().to("mock:domestic");
        });
        AdviceWith.adviceWith(camelContext, "international-handler", route -> {
            route.weaveAddLast().to("mock:international");
        });
        AdviceWith.adviceWith(camelContext, "hazmat-handler", route -> {
            route.weaveAddLast().to("mock:hazmat");
        });
    }

    @BeforeEach
    void resetMocks() {
        MockEndpoint.resetMocks(camelContext);
    }

    @Test
    void domesticOrderRoutesToDomestic() throws Exception {
        MockEndpoint domestic = camelContext.getEndpoint("mock:domestic", MockEndpoint.class);
        MockEndpoint international = camelContext.getEndpoint("mock:international", MockEndpoint.class);
        MockEndpoint hazmat = camelContext.getEndpoint("mock:hazmat", MockEndpoint.class);

        domestic.expectedMessageCount(1);
        international.expectedMessageCount(0);
        hazmat.expectedMessageCount(0);

        String order = """
            {"order_id": 1001, "country": "US", "shipping_type": "STANDARD", "amount": 59.99}
            """;
        producer.sendBody("direct:validate-order", order);

        domestic.assertIsSatisfied();
        international.assertIsSatisfied();
        hazmat.assertIsSatisfied();
    }
}
```

Notice how the test asserts that the *other* mocks received zero messages. This is the unit-test equivalent of negative testing — proving that the domestic order did *not* go to the international or hazmat handlers. Each branch gets its own test method with appropriate assertions on all three mocks.

# Mocking external services

When a route calls an external service, `@InjectMock` (from Quarkus) or `@MockBean` (from Spring Boot) lets you stub it:

```java
@QuarkusTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class OrderEnrichmentRouteTest {

    @Inject
    CamelContext camelContext;

    @Inject
    ProducerTemplate producer;

    @InjectMock
    InventoryService inventoryService;

    @BeforeEach
    void setup() {
        MockEndpoint.resetMocks(camelContext);

        when(inventoryService.checkStock(any())).thenReturn(Map.of(
            "order_id", 3001,
            "item_sku", "ELEC-TV-55",
            "warehouse", "WAREHOUSE-MOCK",
            "stock_available", 99,
            "weight_kg", 15.0
        ));
    }

    @Test
    void orderIsEnrichedWithInventoryData() throws Exception {
        MockEndpoint mock = camelContext.getEndpoint("mock:enriched", MockEndpoint.class);
        mock.expectedMessageCount(1);

        String order = """
            {"order_id": 3001, "item_sku": "ELEC-TV-55", "amount": 599.99}
            """;
        producer.sendBody("direct:enrich-order", order);

        mock.assertIsSatisfied();

        String body = mock.getReceivedExchanges().get(0).getIn().getBody(String.class);
        assertTrue(body.contains("WAREHOUSE-MOCK"));
    }
}
```

The Mockito stub replaces the real `InventoryService` with a predictable response. 
The test verifies that the route correctly calls the service and includes the enrichment data in the output. 
This is useful for services that are expensive, slow, or unavailable in the test environment.

# Integration testing with Citrus

Citrus integration tests operate at a different level. They start real infrastructure, send messages through real Kafka brokers, and validate the output on real topics. 
Here is what the filter route test looks like with Citrus (from the [Routing Fundamentals example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/09-routing-fundamentals/quarkus)):

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
    class MessageFilterTest {

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
    }
}
```

The Citrus test sends a JSON message to the real `eip.orders.placed` Kafka topic. 
The Camel route — running inside the Quarkus application — consumes from that topic, applies the filter, and publishes to `eip.orders.high-value`. 
The test then reads from `eip.orders.high-value` and validates the entire message body against the template.

The negative test uses `expectTimeout()` — it sends a low-value order and verifies that nothing arrives on the output topic within 5 seconds. 
This is a stronger guarantee than `mock.expectedMessageCount(0)` because it tests the full pipeline, including Kafka serialization and consumer group behavior.

## What Citrus catches that unit tests miss

The Citrus test exercises the entire transport layer. Consider these failure modes:

- **Missing `marshal().json()`** — the route unmarshals JSON into a Java Map for filtering but forgets to marshal it back to JSON before sending to Kafka. The Kafka topic receives `{order_id=2001, amount=250.0}` instead of valid JSON. The unit test with `MockEndpoint` passes because it captures the Java object. The Citrus test fails because it reads the actual bytes off the topic and validates them as JSON.

- **Kafka header loss** — the route strips or overwrites Kafka headers during processing. The unit test never sees headers because `ProducerTemplate` doesn't simulate Kafka header behavior. The Citrus test validates headers explicitly with `.header("kafka.KEY", "${id}")`.

- **Consumer group misconfiguration** — the route uses a consumer group that conflicts with another route, causing messages to be split between consumers. This is invisible in unit tests because there is no real consumer group. Citrus tests with unique consumer groups detect this immediately.

# When to use which

Here is a practical decision framework:

| Concern                        | Unit Test (AdviceWith)    | Citrus Integration Test          |
|--------------------------------|---------------------------|----------------------------------|
| Routing logic (choice, filter) | Yes                       | Yes                              |
| Transformation correctness     | Yes                       | Yes                              |
| Bean/service method behavior   | Yes (with mocks)          | Yes (with real services)         |
| Serialization format on wire   | No                        | Yes                              |
| Kafka header propagation       | No                        | Yes                              |
| Consumer group behavior        | No                        | Yes                              |
| Multi-hop async flows          | No                        | Yes                              |
| REST endpoint HTTP behavior    | Partial (via RestAssured) | Yes (via HTTP actions)           |
| Execution speed                | Fast (milliseconds)       | Slower (seconds, Docker startup) |
| Infrastructure needed          | None                      | Docker, Testcontainers           |
| Failure isolation              | Precise (single route)    | Broader (full pipeline)          |

**Start with unit tests** for every route's core logic. They run fast, give precise failure messages, and catch most predicate and transformation bugs during development.

**Add Citrus integration tests** for end-to-end flows that cross transport boundaries. 
If a message goes from Kafka to your route to another Kafka topic — or from a REST endpoint through internal processing to a database — that flow needs an integration test. 
The transport layer is where the most painful production bugs hide.

# Combining both in a single project

The [Testing Strategies example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/37-testing-strategies/quarkus) demonstrates this layering of unit and integration tests in one Maven project.

The separation of test categories is not just organizational. 
Maven's surefire plugin runs the unit tests (`*Test.java`) during the `test` phase, while the failsafe plugin runs the integration tests (`*IT.java`) during the `integration-test` phase. 
This means `mvn test` gives fast feedback during development, and `mvn verify` runs the full suite including integration tests.

The `IntegrationTestProfile` is critical for the Quarkus integration tests. 
It disables Kafka Dev Services (since the integration tests manage their own infrastructure or use AdviceWith to bypass Kafka) and sets the payment gateway to mock mode:

```java
public class IntegrationTestProfile implements QuarkusTestProfile {

    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of(
            "payment.gateway.mode", "mock",
            "quarkus.kafka.devservices.enabled", "false",
            "quarkus.http.test-port", "0"
        );
    }
}
```

# Designing routes for testability at both levels

The `OrderFilterRoute` shown earlier illustrates an important design principle: separate the transport layer from the business logic. 
The core filter logic lives in `direct:filter-order` — a `direct:` endpoint that can be called from a unit test without any transport infrastructure. 
The Kafka consumer is a thin adapter that delegates to the same `direct:` endpoint.

This pattern makes the route testable at both levels:

- **Unit test**: call `direct:filter-order` with `ProducerTemplate`, assert with `MockEndpoint` — no Kafka needed.
- **Integration test**: send to the real Kafka topic, the Kafka consumer route calls `direct:filter-order`, and the output appears on the output topic.

Both tests exercise the same filter predicate, but they do so through different entry points. 
The unit test is fast and precise. 
The integration test is slower but covers the full transport chain.

# Where to find the examples

The complete source code for all tests shown in this post is available in the [EIP with Camel](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing) repository:

- **Testing Strategies example**: [Unit tests](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/37-testing-strategies/quarkus/src/test/java/com/example/eip/testing/unit) and [integration tests](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/37-testing-strategies/quarkus/src/test/java/com/example/eip/testing/integration) side by side.
- **Routes under test**: [OrderFilterRoute](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/37-testing-strategies/quarkus/src/main/java/com/example/eip/testing/OrderFilterRoute.java), [OrderValidationRoute](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/37-testing-strategies/quarkus/src/main/java/com/example/eip/testing/OrderValidationRoute.java), [PaymentGatewayRoute](https://github.com/christophd/eip-with-camel/blob/chore/citrus-testing/examples/37-testing-strategies/quarkus/src/main/java/com/example/eip/testing/PaymentGatewayRoute.java).
- **Citrus integration test examples**: [Routing Fundamentals](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/09-routing-fundamentals/quarkus/src/test/java/com/example/eip/routing) for the full Citrus approach with real Kafka infrastructure.

Use unit tests for fast iteration on routing logic. Add Citrus integration tests for end-to-end confidence across transport boundaries. 
Together they cover the full spectrum — from predicate correctness to wire-format fidelity.

Give it a try, and let us know what you think!
