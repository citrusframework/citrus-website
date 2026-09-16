---
layout: sample
title: Testing the Splitter Pattern with Citrus
name: splitter
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Splitter EIP in Apache Camel with Citrus across Quarkus, Spring Boot and YAML DSL
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/splitter/
---

Many integration scenarios deal with individual messages — one order, one event, one notification.
At the same time a system often encounters composite messages: a bulk order with dozens of line items, a CSV file with hundreds of records, or an API response containing a list of resources.
Processing the entire composite as a single unit is fragile.
A failure on one item can block or roll back everything else, and different items may need entirely different processing paths.

The [Splitter](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Sequencer.html) pattern, described in *Enterprise Integration Patterns* by Hohpe and Woolf, solves this by breaking a composite message into its individual parts and routing each part independently.
Each part becomes its own exchange — its own message with its own lifecycle.
A failure on item 47 out of 50 does not affect the other 49.

[Apache Camel](https://camel.apache.org) implements this pattern with the `split()` EIP, which takes an expression — JSONPath, XPath, tokenizer, or custom logic — and produces one exchange per element.
The implementation is concise, but testing it raises questions that don't come up with single-message patterns.
When a batch of two items enters the splitter, exactly two individual messages must appear on the output channel — not one, not three.
Each output message must represent one item from the original batch, and the content must survive the split intact.

This is where [Citrus](https://citrusframework.org) comes in.
Citrus can send a composite message to the input topic, then consume and validate each individual message from the output topic — proving that the splitter produced the correct number of parts with the expected content.
In this post, we walk through a complete example that tests a Splitter route with Citrus, running on Quarkus, Spring Boot, and the Camel YAML DSL.

# The Camel route under test

The scenario is a batch order processor.
Orders arrive on a Kafka topic `eip.orders.batch` as JSON messages.
Each message contains a `batch_id` and an `items` array with one or more line items.
The splitter extracts each item from the array and publishes it as a separate message on the `eip.orders.individual` topic.

Here is the Camel route on Quarkus:

```java
@ApplicationScoped
public class SplitterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.batch?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=splitter-demo")
            .routeId("order-splitter")
            .unmarshal().json()
            .log("Batch received with ${body[items].size()} items")
            .split(jsonpath("$.items[*]"))
                .log("Processing item: ${body[item_sku]} qty=${body[quantity]}")
                .marshal().json()
                .to("kafka:eip.orders.individual?brokers={% raw %}{{kafka.brokers}}{% endraw %}}")
            .end();
    }
}
```

There are a few things worth noting about this route.

First, the `split(jsonpath("$.items[*]"))` expression targets the `items` array inside the JSON body.
Camel evaluates this JSONPath against the unmarshalled message and produces one exchange per array element.
Each exchange's body is set to the individual item object — not the original batch message.
This means downstream processors see `{"item_sku": "PART-A1", "quantity": 2}`, not the entire batch wrapper.

Second, the route marshals each split item back to JSON before publishing to the output topic.
After the `split()` EIP extracts an item, the body is a Java `Map`.
The downstream Kafka consumers expect JSON, so `marshal().json()` serializes each item before it reaches the `to()` endpoint.

Third, everything inside the `split()` block runs once per item.
The `log()` call inside the block fires for every item in the batch, and the `to()` publishes each item individually.
The `end()` marks where the split processing ends — any steps after `end()` would execute once, after all split items have been processed.

On Spring Boot, the route is identical except for the annotation:

```java
@Component
public class SplitterRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.batch?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=splitter-demo")
            .routeId("order-splitter")
            .unmarshal().json()
            .log("Batch received with ${body[items].size()} items")
            .split(jsonpath("$.items[*]"))
                .log("Processing item: ${body[item_sku]} qty=${body[quantity]}")
                .marshal().json()
                .to("kafka:eip.orders.individual?brokers={% raw %}{{kafka.brokers}}{% endraw %}}")
            .end();
    }
}
```

The same routing logic expressed in the Camel YAML DSL:

```yaml
- route:
    id: order-splitter
    from:
      uri: "kafka:eip.orders.batch"
      parameters:
        brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
        groupId: splitter-demo
      steps:
        - unmarshal:
            json:
              library: Jackson
        - log: "Batch received with ${body[items].size()} items"
        - split:
            jsonpath: "$.items[*]"
            steps:
              - log: "Processing item: ${body[item_sku]} qty=${body[quantity]}"
              - marshal:
                  json:
                    library: Jackson
              - to:
                  uri: "kafka:eip.orders.individual"
                  parameters:
                    brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
```

All three variants express the same behavior: receive a batch, extract each item, publish each item individually.

# What makes testing a splitter different

Testing a splitter is fundamentally different from testing a router or a filter.
With a Content-Based Router, one message goes in and one message comes out — you just need to verify it lands on the right topic.
With a Message Filter, one message goes in and either one or zero messages come out.
But with a Splitter, one message goes in and *N* messages come out, where *N* depends on the content of the input message.

This creates two distinct verification challenges.

**Cardinality**: the test must prove that exactly the right number of messages were produced.
If the batch contains two items, two messages must appear on the output topic — not one (the splitter skipped an item), and not three (the splitter duplicated an item or leaked the wrapper).

**Content isolation**: each output message must contain only its individual item, not the entire batch or a corrupted fragment.
The split expression extracts array elements, but if the expression is wrong — say `$.items` instead of `$.items[*]` — the splitter might emit the entire array as one message instead of splitting it into parts.

Citrus addresses both challenges by consuming multiple messages from the output topic and validating each one against a template that describes what an individual item should look like.

# The message templates

The test uses two JSON templates.
The first is the batch order sent to the input topic, stored in `src/test/resources/templates/batch-order.json`:

```json
{
  "batch_id": "BATCH-${id}",
  "items": [
    { "item_sku": "PART-A1", "quantity": 2 },
    { "item_sku": "PART-B2", "quantity": 1 }
  ]
}
```

The `${id}` placeholder is resolved by Citrus at runtime from test variables, producing a unique batch ID for each test run.
The batch contains two items with known SKUs and quantities — specific enough to verify that the splitter preserves content, but simple enough to keep the test readable.

The second template describes what an individual item should look like after splitting, stored in `src/test/resources/templates/item.json`:

```json
{
  "item_sku": "${item_sku}",
  "quantity": "@ignore@"
}
```

This template uses test variables and Citrus's `@ignore@` validation matcher.
The `@ignore@` marker tells Citrus to accept any value for that field — it only checks that the field exists and has the right JSON structure.
The template asserts the message *structure* (has `item_sku` and `quantity` fields) without pinning the specific values.
This is a deliberate choice: the test focuses on verifying that the splitter produces the correct *number* of messages with the correct *structure*, rather than asserting exact quantity values in each item.

_TIP:_ If you need to validate specific field values, you can use separate templates for each split item or use more test variables in the template.
For many splitter tests, though, structural validation is sufficient — the important thing is that two messages arrived, both with the expected fields.

# Testing that a batch splits into individual items

The core test sends a batch with two items and verifies that exactly two individual messages appear on the output topic.
Here is the test on Quarkus:

```java
@Nested
class SplitterTest {

    @Test
    public void shouldSplitBatchIntoIndividualItems() {
        t.given(
            createVariables()
                .variable("id", "citrus:randomNumber(4)")
        );

        t.given(waitForCamelRouteStarted("order-splitter", camelContext));

        t.when(
            send()
                .endpoint("kafka:eip.orders.batch")
                .message()
                .body(Resources.create("templates/batch-order.json"))
                .header("kafka.KEY", "BATCH-${id}")
        );

        t.then(
            createVariables()
                .variable("item_sku", "PART-A1"),
            receive()
                .endpoint("kafka:eip.orders.individual?consumerGroup=citrus-individual-group")
                .message()
                .body(Resources.create("templates/item.json"))
        ).and(
            createVariables()
                .variable("item_sku", "PART-B2"),
            receive()
                .endpoint("kafka:eip.orders.individual?consumerGroup=citrus-individual-group")
                .message()
                .body(Resources.create("templates/item.json"))
        );
    }
}
```

The test follows Citrus's given-when-then structure.

The `given` phase creates a random `id` variable and waits for the `order-splitter` route to be fully started and connected to Kafka.
The `waitForCamelRouteStarted` utility (described below) uses Camel's Control Bus to poll the route status, preventing the test from sending messages before the route's consumer is ready.

The `when` phase sends the batch order to the `eip.orders.batch` input topic.
The message body is loaded from the `batch-order.json` template, which Citrus resolves with the `${id}` variable.
The Kafka key is set to `BATCH-${id}` to ensure consistent partitioning for the batch.

The `then` phase is where the splitter-specific verification happens.
The test executes *two* consecutive `receive()` actions on the `eip.orders.individual` output topic.
Both receives use the **same consumer group** (`citrus-individual-group`). This is intentional — we want the second receive to pick up where the first left off. If they used different consumer groups, both would try to read from the beginning of the topic, and Kafka's partition assignment might give both the same message.

This structure is the key to testing a splitter: the number of `receive()` actions matches the expected number of split items.
If the splitter produces only one message, the second `receive()` will time out and the test fails.
If the splitter produces three messages, the test passes but leaves an unconsumed message — which could be caught by a more defensive test that adds an `expectTimeout` after the two receives.

# Setting up the test infrastructure

The route consumes from Kafka, so the test needs a running broker.
Citrus manages this lifecycle with Testcontainers — a Docker Compose file defines the Kafka setup, and Citrus starts it before the test suite and tears it down after.

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

# Waiting for the Camel route

Camel routes that consume from Kafka take a moment to start and subscribe to the broker.
If the test sends a message before the route's consumer is ready, the message sits unprocessed and the test times out.

The example addresses this with a reusable utility interface that uses Camel's Control Bus to poll the route status:

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

This polls the Camel context up to 20 times with one-second pauses.
Once the route reports `Started`, it waits an additional five seconds for the Kafka consumer to fully connect.
Every test class implements this interface and calls `waitForCamelRouteStarted(...)` in its `given` phase.

# Testing with the YAML DSL

The Camel CLI supports running routes from YAML files and testing them with Citrus YAML test definitions.
The Splitter test in YAML DSL covers the same scenario — sending a batch and verifying the individual items:

```yaml
name: splitter-test
description: >-
  Test verifying the Splitter pattern — batch order is split
  into individual item messages
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
            name: "order-splitter"
            file: "../splitter.yaml"
            systemProperties:
              file: "../application.properties"

  - createVariables:
      variables:
        - name: id
          value: "citrus:randomNumber(4)"
  - send:
      endpoint: >-
        kafka:eip.orders.batch?server=${kafka.broker}
      message:
        body:
          resource:
            file: "templates/batch-order.json"
  - createVariables:
      variables:
        - name: item_sku
          value: PART-A1
  - receive:
      endpoint: >-
        kafka:eip.orders.individual?server=${kafka.broker}&consumerGroup=citrus-individual-group
      message:
        body:
          resource:
            file: "templates/item.json"
  - createVariables:
      variables:
        - name: item_sku
          value: PART-B2
  - receive:
      endpoint: >-
        kafka:eip.orders.individual?server=${kafka.broker}&consumerGroup=citrus-individual-group
      message:
        body:
          resource:
            file: "templates/item.json"
  - camel:
      jbang:
        verify:
          integration: "order-splitter"
          logMessage: "Processing item: PART-A1"
  - camel:
      jbang:
        verify:
          integration: "order-splitter"
          logMessage: "Processing item: PART-B2"
```

The YAML test is self-contained: it starts the Kafka broker with Testcontainers, launches the Camel integration with `camel:jbang:run`, sends the batch, and verifies the output.

Notice the `camel:jbang:verify` actions at the very end.
These check the running integration's log output for the expected processing messages — `"Processing item: PART-A1"` and `"Processing item: PART-B2"`.
This serves as a secondary verification that the splitter actually processed both items, independent of what appears on the output topic.

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
    class SplitterTest {
        // ... test methods
    }
}
```

On Spring Boot, the test uses `@CitrusSpringSupport` alongside Spring Boot and Camel test annotations:

```java
@SpringBootTest(classes = RoutingFundamentalsApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class SplitterTest {

        @CitrusResource
        TestCaseRunner t;

        // ... test methods
    }
}
```

On Spring Boot, the `TestCaseRunner` is declared in the `@Nested` inner class, while on Quarkus it lives at the outer class level — a structural difference driven by how each test framework manages injection scopes.
The actual test methods inside `SplitterTest` are identical on both runtimes.

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
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-validation-json</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

The `citrus-camel` module provides the Control Bus integration used in `waitForCamelRouteStarted`.
The `citrus-kafka` module provides the Kafka send and receive actions.
The `citrus-testcontainers` module manages the Docker Compose lifecycle.
The `citrus-validation-json` module enables JSON message validation with the template-based assertions and `@ignore@` matchers.

# Running the tests

Make sure Docker (or Podman) is running, then execute:

```bash
# Quarkus
cd examples/09-routing-fundamentals/quarkus
mvn verify

# Spring Boot
cd examples/09-routing-fundamentals/spring-boot
mvn verify
```

The test suite starts the Kafka broker via Testcontainers, boots the Camel application, runs the splitter test, and tears everything down.

For the YAML DSL variant, install the [Camel CLI](https://camel.apache.org/manual/camel-jbang.html) and run:

```bash
cd examples/09-routing-fundamentals/yaml-dsl
camel test test/splitter.citrus.it.yaml
```

# Key takeaways

Testing a Splitter means verifying that a composite message is decomposed into the correct number of parts, each with the expected structure.
Unlike single-message patterns where you check *which* output channel a message lands on, a Splitter test must verify *how many* messages are produced and *what* each one contains.

Here is what Citrus brings to this problem:

- **Multiple receives for cardinality** — Two `receive()` actions inside the same `repeatOnError` block prove that exactly two individual messages were produced from a two-item batch. The number of receives matches the expected split count.
- **Test Variables** enable templates to add placeholders that are set by the test before validation takes place. This makes the same verification template configurable for different items. 
- **`@ignore@` for structural validation** — The `item.json` template uses `@ignore@` to verify that each output message has the correct JSON structure without asserting exact field values. This makes the test resilient to message ordering across Kafka partitions.
- **Log verification in YAML DSL** — The `camel:jbang:verify` action checks the running integration's log for processing messages, providing a secondary confirmation that the splitter handled each item.
- **Multi-runtime portability** — The same test logic runs on Quarkus, Spring Boot, and the Camel CLI with YAML DSL. Only the wiring annotations change; the send, receive, and validation logic stays identical.

The complete source code for this example is available on GitHub:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/spring-boot)
- [YAML DSL variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/09-routing-fundamentals/yaml-dsl)

Give it a try, and let us know what you think!
