---
layout: post
title: Multi-Runtime Testing — One Test Pattern, Three Different Runtimes
short-title: Multi-Runtime Testing
author: Christoph Deppisch
github: christophd
categories: [blog]
---

[Apache Camel](https://camel.apache.org) runs on multiple runtimes like Quarkus, Spring Boot, and standalone with YAML DSL via JBang. 
Your routes might be identical across runtimes — the same Kafka consumer, the same content-based router, the same message filtering or database enrichment. 
Depending on the underlying framework our tests need runtime-specific wiring: different annotations, different dependency injection, different lifecycle management. The good news is that the core [Citrus](https://citrusframework.org) testing patterns are the same everywhere. Only the scaffolding changes.

In this post, we walk through the same test scenario — send an order to Kafka, verify that events get produced on the correct output topic — implemented on all three runtimes Quarkus, Spring Boot and plain YAML. We highlight exactly what changes between Quarkus, Spring Boot, and YAML DSL, and what stays constant. By the end, you will know how to apply proper Citrus integration testing to the runtime that matches your project.

![Featured](/img/assets/multi-runtime-testing/featured.png){:width="700px" .center-image}
*AI generated with Google Gemini*

# The common test pattern

Regardless of runtime, every Citrus integration test for Camel follows the same pattern:

1. Wait for the Camel route to start.
2. Send a message to the route's input endpoint.
3. Receive and verify the message on the route's output endpoint.

Here is what that looks like in Java — this code is identical across Quarkus and Spring Boot runtimes:

```java
t.given(
    createVariables()
        .variable("id", "citrus:randomNumber(4)")
        .variable("amount", 100)
        .variable("status", "placed")
        .variable("priority", "STANDARD")
);

t.given(waitForCamelRouteStarted("datatype-channel-router", camelContext));

t.when(
    send()
        .endpoint("kafka:eip.orders.placed")
        .fork(true)
        .message()
        .body(Resources.create("templates/order.json"))
        .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
);

t.then(
    receive()
        .endpoint("kafka:eip.orders.placed.typed?consumerGroup=citrus-placed-group-group")
        .message()
        .body(Resources.create("templates/order.json"))
        .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
);
```

The test actions — `send()`, `receive()`, `repeatOnError()`, `waitForCamelRouteStarted()`, `createVariables()` — are runtime-agnostic. They come from the `TestActionSupport` interface, which both Quarkus and Spring Boot test classes can implement in the exact same way. Both test classes in the different runtimes may implement the exact same base interface `EipTestSupport` that we explore next.

## The message template

Place an `order.json` template under `src/test/resources/templates/`:

```json
{
  "order_id": ${id},
  "customer_id": "CUST-00${id}",
  "item_sku": "SKU-SHIP-${id}",
  "quantity": 1,
  "amount": ${amount},
  "status": "${status}",
  "shipping_priority": "${priority}"
}
```

Citrus resolves the `${...}` placeholders at runtime. The same template is used for both sending and receiving — the test sends an order with specific values and expects to receive it back with the same values after processing. This ensures the route preserved the message content faithfully.

Test variable support is runtime agnostic and the same template can be used across all different runtimes Quarkus, Spring Boot and YAML DSL.

## Shared test actions - Waiting for Camel routes to start

Test classes in all runtimes can implement the base interface `TestActionSupport` with all its basic Citrus test actions like `send()` and `receive()`. In fact, we can use a custom base interface `EipTestSupport` that extends the Citrus base interface. This is the place to provide default shared test action methods that can be used in all tests, like the `waitForCamelRouteStarted()` helper method that makes use of the normal Citrus `repeatOnError()` to poll the Camel control bus until a given route reaches `Started` status. The helper method only depends on the `CamelContext`, which both runtimes provide — just through different injection mechanisms.

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
                            .result(ServiceStatus.Started)
                            .description("Waiting for Camel route '%s' to be started ..."
                                .formatted(routeId)),
                    sleep().seconds(5)
                );
    }
}
```

This uses Camel's **ControlBus** component to query the route's status. The ControlBus is a management interface built into Camel that lets you inspect and control routes at runtime. The method polls the route status in a retry loop — if the route hasn't started yet (because the Kafka broker is still initializing, for example), it retries up to 20 times with 1-second intervals.

The test class implements this interface (`class EipTests implements EipTestSupport`), which makes the method available in all test methods. This interface is identical across chapters and can be copied as-is into any Citrus test package.

# Quarkus runtime

The Quarkus runtime uses `@QuarkusTest` for the test lifecycle and `@CitrusSupport` (coming from the Citrus Quarkus extension) to activate Citrus. Here is the full test class structure:

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
    class DatatypeChannelRouteTest {

        @Test
        public void shouldHandlePlacedOrders() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("status", "placed")
            );
            
            t.given(waitForCamelRouteStarted("datatype-channel-router", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.placed")
                    .fork(true)
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.placed.typed?consumerGroup=citrus-placed-group-group")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );
        }
    }
}
```

Three things to notice:

## TestCaseRunner placement

The `@CitrusResource TestCaseRunner t` field is declared on the **outer class**, not inside each `@Nested` class. Quarkus injects the runner at the top level, and nested test classes inherit it. This is the opposite of Spring Boot, where the runner must be declared inside each nested class.

## CamelContext injection

The `CamelContext` requires **both** `@Inject` and `@BindToRegistry`:

```java
@Inject
@BindToRegistry
CamelContext camelContext;
```

`@Inject` is the standard CDI injection. `@BindToRegistry` registers the context in the Citrus bean registry so that Citrus test actions can find it. Without `@BindToRegistry`, actions like `camel().camelContext(camelContext)` would fail to resolve the context.

## Infrastructure setup with @CitrusConfiguration

Quarkus uses Citrus-native annotations for infrastructure setup:

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

The class uses `@CitrusConfiguration` (not Spring's `@Configuration`) and `@BindToRegistry` (not Spring's `@Bean`). Citrus discovers this configuration class through a properties file.

## Configuration discovery

Quarkus requires a `citrus-application.properties` file in `src/test/resources/` to tell Citrus where to find the configuration:

```properties
citrus.java.config=com.example.eip.channels.config.EipInfraSetup
```

Without this file, Citrus will not discover the `EipInfraSetup` class and the Docker Compose infrastructure will not start.

## Dependencies

The Quarkus test module requires these Citrus dependencies:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-camel</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-junit-jupiter</artifactId>
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
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-validation-json</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

The key module is `citrus-quarkus` — it provides the `@CitrusSupport` annotation and the Quarkus-specific test lifecycle integration.

You can find the complete Quarkus test in the [04-channel-types/quarkus example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/04-channel-types/quarkus).

# Spring Boot runtime

The Spring Boot runtime uses a different set of annotations but the same core test logic. Here is the test class structure:

```java
@SpringBootTest(classes = ChannelTypesApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class DatatypeChannelRouteTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldHandlePlacedOrders() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("status", "placed")
            );
            
            t.given(waitForCamelRouteStarted("datatype-channel-router", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.placed")
                    .fork(true)
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.placed.typed?consumerGroup=citrus-placed-group-group")
                    .message()
                    .body(Resources.create("templates/order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );
        }
    }
}
```

The test body inside `shouldHandlePlacedOrders` is identical to the Quarkus version. Everything that differs is in the class-level annotations and field declarations.

## Class-level annotations in Spring Boot

Spring Boot requires four class-level annotations to combine Spring, Apache Camel and Citrus:

- `@SpringBootTest(classes = ChannelTypesApplication.class)` — boots the Spring application context.
- `@CamelSpringBootTest` — integrates the Camel test lifecycle with Spring Boot.
- `@CitrusSpringSupport` — activates Citrus with Spring-aware dependency injection.
- `@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })` — loads the infrastructure setup and Citrus Spring configuration.

The `@ContextConfiguration` is how Spring Boot discovers the infrastructure setup class — no `citrus-application.properties` needed.

## TestCaseRunner placement — the key difference

In Spring Boot, `@CitrusResource TestCaseRunner t` must be declared inside **each `@Nested` class**, not on the outer class:

```java
@Nested
class DatatypeChannelRouteTest {

    @CitrusResource
    TestCaseRunner t;

    // tests use t here
}

@Nested
class PointToPointRouteTest {

    @CitrusResource
    TestCaseRunner t;

    // tests use t here
}
```

This is the single most common source of confusion when moving tests between runtimes. In Quarkus, declaring `t` on the outer class works because the CDI-based injection flows into nested classes. In Spring Boot, each nested class gets its own Citrus test context, so each needs its own runner.

## CamelContext injection — simpler

Spring Boot uses standard `@Autowired` without any registry annotation:

```java
@Autowired
CamelContext camelContext;
```

No `@BindToRegistry` is needed because Spring Boot's application context already makes the `CamelContext` available to Citrus through Spring's dependency injection.

## Infrastructure setup with @Configuration

Spring Boot uses Spring-native annotations:

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

Compare this to the Quarkus version: `@Configuration` replaces `@CitrusConfiguration`, and `@Bean` replaces `@BindToRegistry`. The method bodies are identical — only the annotations differ.

## Dependencies

The Spring Boot module swaps `citrus-quarkus` for `citrus-spring`:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-spring</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

All other Citrus dependencies (`citrus-camel`, `citrus-junit-jupiter`, `citrus-kafka`, `citrus-testcontainers`, `citrus-validation-json`) are the same.

You can find the complete Spring Boot test in the [04-channel-types/spring-boot example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/04-channel-types/spring-boot).

# YAML DSL runtime

The YAML DSL runtime takes a completely different approach. There are no Java classes, no annotations, and no POM dependencies. Tests are written as `.citrus.it.yaml` files and executed by the Camel JBang test runner.

Here is a YAML DSL test for the same datatype channel scenario:

```yaml
name: datatype-channel-test
description: Test verifying the Datatype Channel route
variables:
  - name: kafka.broker
    value: localhost:9092
actions:
  - testcontainers:
      compose:
        up:
          file: "_infra/compose.yaml"
  - camel:
      jbang:
        run:
          integration:
            name: "datatype-channel"
            file: "../datatype-channel.yaml"
            systemProperties:
              file: "../application.properties"
  - createVariables:
      variables:
        - name: id
          value: "citrus:randomNumber(4)"
        - name: status
          value: "placed"
  - send:
      endpoint: >-
        kafka:eip.orders.placed?server=${kafka.broker}
      message:
        body:
          resource:
            file: "templates/order.json"
  - camel:
      jbang:
        verify:
          integration: "datatype-channel"
          logMessage: "→ eip.orders.placed.typed"
```

## Structure

A YAML DSL test file has three top-level keys:

- `name` and `description` — test metadata.
- `variables` — test-scoped variables, equivalent to `createVariables()` in Java.
- `actions` — the test steps, executed in order.

## Starting infrastructure and routes

The first two actions handle what the `EipInfraSetup` class does in Java:

1. `testcontainers.compose.up` starts Docker Compose infrastructure.
2. `camel.jbang.run` starts the Camel route as a JBang integration, loading its properties file.

There is no equivalent of `BeforeSuite`/`AfterSuite` — the infrastructure starts inline as the first test actions.

## Verification with log messages

Instead of receiving messages from Kafka output topics, YAML DSL tests use `camel.jbang.verify` to check route log output:

```yaml
- camel:
    jbang:
      verify:
        integration: "datatype-channel"
        logMessage: "→ eip.orders.placed.typed"
```

This asserts that the running Camel integration logged a message containing the specified text. It is a lighter-weight verification than consuming from the output topic, and it works well for routes where the log statement confirms the routing decision.

## Configuration

YAML DSL tests has a `citrus-application.properties` in the test directory with some properties specific to the JBang runtime:

```properties
citrus.camel.jbang.dump.integration.output=true
citrus.camel.jbang.version=4.21.0
```

The `dump.integration.output=true` setting enables log capture so that `camel.jbang.verify` can match against route log output. The `version` setting pins the JBang Camel version.

You can find the complete YAML DSL tests in the [04-channel-types/yaml-dsl example](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing/examples/04-channel-types/yaml-dsl).

# Side-by-side comparison

Here is the full comparison of what changes across runtimes:

| Aspect                   | Quarkus                                | Spring Boot                                                                                | YAML DSL                        |
|--------------------------|----------------------------------------|--------------------------------------------------------------------------------------------|---------------------------------|
| Test annotations         | `@QuarkusTest`, `@CitrusSupport`       | `@SpringBootTest`, `@CamelSpringBootTest`, `@CitrusSpringSupport`, `@ContextConfiguration` | N/A                             |
| TestCaseRunner injection | Outer class                            | Each `@Nested` class                                                                       | N/A (implicit)                  |
| CamelContext injection   | `@Inject @BindToRegistry`              | `@Autowired`                                                                               | N/A                             |
| Infrastructure setup     | `@CitrusConfiguration @BindToRegistry` | `@Configuration @Bean`                                                                     | `testcontainers` action         |
| Config discovery         | `citrus-application.properties`        | Automatic via `@ContextConfiguration`                                                      | `citrus-application.properties` |
| Runtime module           | `citrus-quarkus`                       | `citrus-spring`                                                                            | Built into JBang CLI tooling    |
| Verification             | Kafka event receive                    | Kafka event receive                                                                        | Kafka event + Log message       |

The key takeaway: the test actions — `send()`, `receive()`, `repeatOnError()`, `waitForCamelRouteStarted()` — are identical in Quarkus and Spring Boot. The YAML DSL uses its own action syntax but expresses the same concepts. What changes is the wiring: how you start the test, how you inject dependencies, and how you discover configuration.

# Infrastructure setup — what is shared

Despite the annotation differences, the infrastructure setup logic is identical across runtimes.

## Docker Compose files

The `compose.yaml` file under `_infra/` is the same for all three runtimes. 
It defines the Kafka broker, schema registry, PostgreSQL, or whatever services the routes need. The only difference is how the test references it:

- **Quarkus/Spring Boot**: `testcontainers().compose().up("_infra/compose.yaml")` in the `EipInfraSetup` class.
- **YAML DSL**: `testcontainers.compose.up.file: "_infra/compose.yaml"` as the first test action.

## Readiness checks

The `waitFor().http()` pattern is the same in both Java runtimes:

```java
waitFor()
    .http()
    .url("http://localhost:8090")
    .seconds(25)
```

In YAML DSL, the route startup is handled by `camel.jbang.run`, which waits for the integration to be ready before proceeding.

## Template files

JSON template files (like `order.json`) live under `src/test/resources/templates/` in Java runtimes and under `test/templates/` in YAML DSL. The file structure is the same — only the variable placeholder style differs (`${id}` vs `${order.id}`).

# Choosing a runtime

Pick the runtime that matches your project:

- **Quarkus** if you are building a Quarkus application. The test annotations are minimal (`@QuarkusTest @CitrusSupport`), and CDI injection keeps the wiring lean.
- **Spring Boot** if you are building a Spring Boot application. The additional annotations (`@CamelSpringBootTest`, `@ContextConfiguration`) are verbose but integrate naturally with Spring's test infrastructure.
- **YAML DSL** if you are prototyping with Camel JBang or writing standalone routes without a Java framework. No POM, no annotations, no compiled code.

The testing patterns transfer across all three. Once you learn `send()`, `receive()`, `repeatOnError()`, and `waitForCamelRouteStarted()` in one runtime, you can apply the same patterns in any other. The EipTestSupport interface, the infrastructure lifecycle, and the assertion strategies are portable — only the annotations and injection mechanisms are runtime-specific.

You can explore all three runtime variants side by side in the [eip-with-camel repository](https://github.com/christophd/eip-with-camel/tree/chore/citrus-testing). Every example under `examples/` has `quarkus/`, `spring-boot/`, and `yaml-dsl/` subdirectories implementing the same routes and tests.

Give it a try, and let us know what you think!
