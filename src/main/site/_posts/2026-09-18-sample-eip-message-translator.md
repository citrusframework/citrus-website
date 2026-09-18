---
layout: sample
title: Testing the Message Translator Pattern with Citrus
name: message-translator
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Message Translator EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/message-translator/
---

Sometimes messages arrive in the format the receiver does not expect.
An external partner sends orders with field names like `orderNumber`, `clientRef`, and `qty`.
Your internal services expect `order_id`, `customer_id`, and `quantity`.
The data is the same, but the shape is different — different field names, different nesting, sometimes different data types.

This mismatch is not a bug.
It is the natural consequence of independent systems evolving on their own schedules with their own domain models.
No two teams name things the same way, and no integration standard has ever changed that.

The [Message Translator](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageTranslator.html) pattern, described by Hohpe and Woolf, solves this by inserting a transformation step between the source and the target.
The translator reads the incoming message, maps its fields to the expected format, and produces a new message that the downstream consumer understands.
It is the most common transformation pattern — almost every integration involves at least one translation.

[Apache Camel](https://camel.apache.org) implements message translation through several mechanisms: `marshal()`/`unmarshal()` for format conversion (XML to JSON, CSV to Avro), `process()` blocks for schema mapping with business logic, and declarative tools like JOLT for JSON-to-JSON transformation.

Testing a message translator is fundamentally about verifying that the mapping is correct.
Send a message in the source format, receive it in the target format, and assert that every field landed where it should.
That sounds simple, but in a real system the translator sits between Kafka topics, interacts with live serialization, and runs inside a framework that manages its lifecycle.
An integration test with [Citrus](https://citrusframework.org) proves that the entire pipeline — Kafka consumer, JSON deserialization, field mapping, JSON serialization, Kafka producer — works end-to-end.

## The scenario

An external partner sends orders to a Kafka topic called `eip.orders.external`.
The orders use the partner's schema: `orderNumber`, `clientRef`, `productCode`, `qty`, `totalValue`, `shipToCountry`, and `hazardous`.

The message translator route reads these orders, maps each field to the internal canonical format (`order_id`, `customer_id`, `item_sku`, `quantity`, `amount`, `destination_country`, `contains_hazmat`), adds a `status` field set to `"NEW"`, and publishes the translated order to `eip.orders.placed`.

## The Camel route

### Quarkus

```java
@ApplicationScoped
public class MessageTranslatorRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.external?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=translator-demo")
            .routeId("message-translator")
            .unmarshal().json()
            .log("External order received: ${body}")
            .process(exchange -> {
                var external = exchange.getIn().getBody(Map.class);
                var canonical = new LinkedHashMap<String, Object>();
                canonical.put("order_id", external.get("orderNumber"));
                canonical.put("customer_id", external.get("clientRef"));
                canonical.put("item_sku", external.get("productCode"));
                canonical.put("quantity", external.get("qty"));
                canonical.put("amount", external.get("totalValue"));
                canonical.put("destination_country",
                    external.getOrDefault("shipToCountry", "US"));
                canonical.put("contains_hazmat",
                    Boolean.TRUE.equals(external.get("hazardous")));
                canonical.put("status", "NEW");
                exchange.getIn().setBody(canonical);
            })
            .log("Translated to canonical: order_id=${body[order_id]}, amount=${body[amount]}")
            .marshal().json()
            .to("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

The route follows three steps:

1. **Deserialize** — `unmarshal().json()` converts the raw Kafka message bytes into a Java `Map`.
2. **Translate** — The `process()` block reads fields from the external schema and writes them into a new `LinkedHashMap` using the internal field names. It also applies light business logic: `shipToCountry` defaults to `"US"` if absent, and the `hazardous` boolean is normalized through `Boolean.TRUE.equals()` to handle nulls safely.
3. **Serialize and publish** — `marshal().json()` converts the canonical map back to JSON, and the `to()` endpoint publishes it to the output topic.

### Spring Boot

The Spring Boot variant is identical except for the class annotation:

```java
@Component
public class MessageTranslatorRoute extends RouteBuilder {

    @Override
    public void configure() {
        from("kafka:eip.orders.external?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=translator-demo")
            .routeId("message-translator")
            .unmarshal().json()
            .log("External order received: ${body}")
            .process(exchange -> {
                var external = exchange.getIn().getBody(Map.class);
                var canonical = new LinkedHashMap<String, Object>();
                canonical.put("order_id", external.get("orderNumber"));
                canonical.put("customer_id", external.get("clientRef"));
                canonical.put("item_sku", external.get("productCode"));
                canonical.put("quantity", external.get("qty"));
                canonical.put("amount", external.get("totalValue"));
                canonical.put("destination_country",
                    external.getOrDefault("shipToCountry", "US"));
                canonical.put("contains_hazmat",
                    Boolean.TRUE.equals(external.get("hazardous")));
                canonical.put("status", "NEW");
                exchange.getIn().setBody(canonical);
            })
            .log("Translated to canonical: order_id=${body[order_id]}, amount=${body[amount]}")
            .marshal().json()
            .to("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

`@Component` replaces `@ApplicationScoped` — the route logic stays identical across both runtimes.

## What makes this testable

A message translator has a clear contract: given input in format A, produce output in format B.
Every field mapping is a discrete assertion.
This makes it an ideal candidate for template-based validation in Citrus.

Instead of writing Java assertions that parse JSON and check individual fields, you define two JSON template files — one for the input (external format) and one for the expected output (canonical format).
The test sends the input template and receives against the output template.
If the translator maps a field incorrectly, renames it wrong, or drops it entirely, the receive action fails with a clear diff between expected and actual.

## Test templates

The input template represents the partner's order format:

```json
{
  "orderNumber": "ORD-${id}",
  "clientRef": "CLIENT-${id}",
  "productCode": "${sku}",
  "qty": ${quantity},
  "totalValue": ${amount},
  "shipToCountry": "${country}",
  "hazardous": ${hazardous}
}
```

The expected output template represents the internal canonical format:

```json
{
  "order_id": "ORD-${id}",
  "customer_id": "CLIENT-${id}",
  "item_sku": "${sku}",
  "quantity": ${quantity},
  "amount": ${amount},
  "destination_country": "${country}",
  "contains_hazmat": ${hazardous},
  "status": "NEW"
}
```

Both templates use the same Citrus variables (`${id}`, `${sku}`, `${quantity}`, etc.).
This is the key insight: the *values* stay the same across the translation — only the *field names* change.
By using identical variables in both templates, the test implicitly asserts that the translator preserves data integrity while remapping the schema.

The `status` field in the output template has no corresponding variable in the input.
It is hardcoded to `"NEW"` because the translator adds it — this verifies that the route correctly injects default values that the external system does not provide.

## Test infrastructure

Both runtimes use a `BeforeSuite`/`AfterSuite` pattern to manage the test infrastructure lifecycle.
The Docker Compose stack includes Kafka and Redis (Redis is used by other transformation patterns in the same example project, but the message translator test only needs Kafka).

### Quarkus infrastructure setup

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

### Spring Boot infrastructure setup

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

The infrastructure code uses Citrus's `testcontainers()` DSL to bring up the Docker Compose stack and `waitFor().http()` to block until the Kafka UI (port 8090) is reachable — a proxy for the Kafka broker itself being ready.
The `afterSuite` stops the Camel context first, then tears down the containers.

The annotation difference is the same as in previous examples: `@CitrusConfiguration` with `@BindToRegistry` for Quarkus, `@Configuration` with `@Bean` for Spring Boot.

## The message translator test

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

This utility polls the Camel route's status via the Control Bus every second, up to 20 retries.
It ensures the `message-translator` route is consuming from Kafka before the test sends any messages.

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
    class MessageTranslatorTest {

        @Test
        public void shouldTranslateExternalOrderToCanonicalFormat() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-ABC-42")
                    .variable("quantity", 5)
                    .variable("amount", 99.95)
                    .variable("country", "US")
                    .variable("hazardous", false)
            );

            t.given(waitForCamelRouteStarted("message-translator", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.external")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/external-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.placed?consumerGroup=citrus-translator-placed-group")
                    .message()
                    .body(Resources.create("templates/canonical-order.json"))
            );
        }
    }
}
```

Let's walk through the given-when-then structure.

**Given — set up variables and wait for the route.**
The test creates six variables that populate both the input and output templates.
`citrus:randomNumber(4)` generates a unique order ID for each test run, preventing cross-test interference on shared Kafka topics.
The `waitForCamelRouteStarted` call blocks until the `message-translator` route reports `Started` via Camel's Control Bus.

**When — send the external order.**
A single message goes to `kafka:eip.orders.external` with the body loaded from `templates/external-order.json`.
Citrus resolves the `${id}`, `${sku}`, `${quantity}`, `${amount}`, `${country}`, and `${hazardous}` placeholders before sending, producing a concrete JSON message in the partner's format.

**Then — receive and validate the canonical order.**
The test consumes from `kafka:eip.orders.placed` using a dedicated consumer group (`citrus-translator-placed-group`) to avoid interfering with the application's own consumers.
The expected body is loaded from `templates/canonical-order.json`, with the same variables resolved.

Citrus performs a structural comparison between the received JSON and the expected template.
If the translator writes `"order_id": "ORD-1234"` but the template expects `"order_id": "ORD-5678"` (because the variables resolve differently), the assertion fails.
If the translator forgets to map `contains_hazmat`, the field is missing from the output and the comparison catches it.

The `fork=true` option on the send operation avoids a blocking situation and handles the asynchronous processing.
The translator route needs time to consume the message, transform it, and publish the result.
While all of that is done the receiving operation initializes the Kafka consumer with a proper offset.
There is no racing condition between the order processing and the consumer initialization.

### Spring Boot test

```java
@SpringBootTest(classes = TransformationApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class MessageTranslatorTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldTranslateExternalOrderToCanonicalFormat() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-ABC-42")
                    .variable("quantity", 5)
                    .variable("amount", 99.95)
                    .variable("country", "US")
                    .variable("hazardous", false)
            );

            t.given(waitForCamelRouteStarted("message-translator", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.external")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/external-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.placed?consumerGroup=citrus-translator-placed-group")
                    .message()
                    .body(Resources.create("templates/canonical-order.json"))
            );
        }
    }
}
```

The test logic is identical.
The differences are confined to the test class annotations and dependency injection:

| Concern                | Quarkus                                | Spring Boot                                |
|------------------------|----------------------------------------|--------------------------------------------|
| Test bootstrap         | `@QuarkusTest`                         | `@SpringBootTest` + `@CamelSpringBootTest` |
| Citrus integration     | `@CitrusSupport`                       | `@CitrusSpringSupport`                     |
| CamelContext injection | `@Inject` + `@BindToRegistry`          | `@Autowired`                               |
| TestCaseRunner scope   | Class-level field                      | Nested class field with `@CitrusResource`  |
| Infrastructure config  | `@CitrusConfiguration` auto-discovered | `@ContextConfiguration` explicit           |

## Why template-based validation beats field-by-field assertions

A common alternative to Citrus's template approach is writing Java assertions that parse the received JSON and check each field individually:

```java
// What you might write without templates
JsonNode result = mapper.readTree(receivedBody);
assertEquals("ORD-1234", result.get("order_id").asText());
assertEquals("CLIENT-1234", result.get("customer_id").asText());
assertEquals("SKU-ABC-42", result.get("item_sku").asText());
// ... and so on for every field
```

This works, but it has three problems:

**1. It doesn't catch extra fields.**
If the translator accidentally passes through a field from the source format (say, `clientRef` alongside `customer_id`), field-by-field assertions won't notice — they only check what they explicitly assert.
Citrus's template comparison validates the entire JSON structure, including the absence of unexpected fields.

**2. It couples the test to the assertion code, not the contract.**
When the schema evolves — a new field is added, a field is renamed — you need to update both the route and the assertion code.
With templates, you update the template file and the test automatically validates against the new contract.

**3. It obscures the mapping.**
Placing the input template and output template side by side immediately shows what changes: `orderNumber` becomes `order_id`, `clientRef` becomes `customer_id`, `qty` becomes `quantity`.
Java assertion code buries this relationship under layers of API calls.

## What the test proves

This single test verifies the complete translation pipeline:

- **Kafka deserialization** — the raw bytes from the external topic are correctly parsed as JSON.
- **Field mapping** — every source field maps to the correct target field (`orderNumber` → `order_id`, `clientRef` → `customer_id`, etc.).
- **Default value injection** — the `status` field is set to `"NEW"` even though it does not exist in the source.
- **Null-safe boolean handling** — the `hazardous` flag is normalized through `Boolean.TRUE.equals()` and mapped to `contains_hazmat`.
- **Kafka serialization** — the canonical map is correctly serialized back to JSON and published to the output topic.
- **End-to-end flow** — the message passes through a real Kafka broker, not a mocked endpoint.

A unit test with mocked Kafka endpoints could verify the `process()` block in isolation.
But it would not catch issues in serialization configuration, Kafka consumer group behavior, or the interaction between `unmarshal().json()` and the downstream `marshal().json()`.
The integration test catches all of these.

## Key takeaways

- **Template pairs are the natural test artifact for translators.** Define the input format in one template and the expected output in another. Shared Citrus variables ensure that values are preserved across the mapping while field names change.
- **`fork=true` on the `send` handles multi-hop async processing.** The translator processes messages asynchronously. Rather than inserting fixed sleeps, Citrus initializes the `receive` immediately until the translated message appears. The non-blocking send operation gives the full pipeline time to complete and the Kafka consumer time to initialize without any racing conditions.
- **`waitForCamelRouteStarted` prevents lost messages.** Messages sent before the Camel route is consuming from Kafka are lost. The Control Bus check ensures the route is ready before the test sends anything.
- **Two runtimes, one test pattern.** The test logic is identical across Quarkus and Spring Boot. Only the bootstrap annotations and dependency injection mechanism differ — the given-when-then structure, the template files, and the Kafka endpoints stay the same.
- **Integration tests catch what unit tests miss.** Serialization mismatches, consumer group configuration, and end-to-end data flow are only visible when the test runs against real infrastructure.
