---
layout: sample
title: Testing the Content Filter Pattern with Citrus
name: content-filter
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Content Filter EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/content-filter/
---

Not every consumer needs every field or data given on the message.

An enriched order flowing through your system might carry a dozen fields: customer ID, product name, product category, weight, shipping zone, hazmat flag, and more.
That is useful for the fulfillment service.
But the analytics pipeline only needs the order ID, the SKU, the quantity, the amount, the destination country, and the status.
More important sensitive data should not be exposed to external services.
Sending the full order to analytics wastes bandwidth, increases storage costs, and — more importantly — risks leaking sensitive data like customer identifiers or internal fields that external systems should never see.

The [Content Filter](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ContentFilter.html) pattern, described by Hohpe and Woolf, strips unwanted fields from a message before forwarding it to a downstream consumer.
It is the inverse of the Content Enricher: where the enricher *adds* data from an external source, the filter *removes* data that the receiver does not need.

Content filters serve four purposes:

- **Security** — Remove internal fields before sending to external systems.
- **Privacy** — Strip personally identifiable information before sending to analytics or logging.
- **Efficiency** — Reduce message size when downstream consumers need only a subset of fields.
- **Compatibility** — Remove fields that an older consumer does not understand.

[Apache Camel](https://camel.apache.org) implements content filtering through `process()` blocks that operate on the message body.
The approach is straightforward — iterate over the fields in the message and keep only the ones that belong in the output.

Testing a content filter is deceptively simple.
The test sends a message with many fields and asserts that the output contains only the allowed ones.
But the real value of an integration test is proving what is *absent*.
A field-by-field assertion that checks only the expected fields will pass even if the filter accidentally lets extra fields through.
A template-based validation in [Citrus](https://citrusframework.org) validates the entire JSON structure, catching both missing fields and unexpected ones.

## The scenario

Enriched orders arrive on the `eip.orders.enriched` topic with the full set of fields: order details, product catalog data, hazmat flags, and customer references.
The content filter route reads each order, applies an allowlist of fields safe for the analytics pipeline, and publishes the stripped-down order to `eip.orders.analytics`.

## Allowlist vs. blocklist

There are two ways to implement a content filter:

- **Allowlist** — Only specified fields pass. Everything else is dropped. New fields added to the source schema are automatically excluded.
- **Blocklist** — Specified fields are removed. Everything else passes through. New fields added to the source schema automatically pass through.

The two approaches have different safety properties.
An allowlist is safer for security-sensitive outputs: you cannot accidentally leak a field you forgot to block, because only explicitly listed fields survive.
A blocklist is more convenient when you want most fields and only need to exclude a few specific ones.

This example uses an allowlist — the right choice when the output goes to an analytics pipeline that should never receive customer identifiers or internal operational fields.

## The Camel route

### Quarkus

```java
@ApplicationScoped
public class ContentFilterRoute extends RouteBuilder {

    private static final Set<String> ALLOWED_FIELDS = Set.of(
        "order_id", "item_sku", "quantity", "amount",
        "destination_country", "shipping_priority", "status"
    );

    @Override
    public void configure() {
        from("kafka:eip.orders.enriched?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=filter-demo")
            .routeId("content-filter")
            .unmarshal().json()
            .log("Filtering PII from order ${body[order_id]}")
            .process(exchange -> {
                var order = exchange.getIn().getBody(Map.class);
                var filtered = new LinkedHashMap<String, Object>();
                for (var entry : ((Map<String, Object>) order).entrySet()) {
                    if (ALLOWED_FIELDS.contains(entry.getKey())) {
                        filtered.put(entry.getKey(), entry.getValue());
                    }
                }
                exchange.getIn().setBody(filtered);
            })
            .log("Filtered: ${body}")
            .marshal().json()
            .to("kafka:eip.orders.analytics?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

The route has three stages:

1. **Deserialize** — `unmarshal().json()` converts the Kafka message bytes into a Java `Map`.
2. **Filter** — The `process()` block iterates over every field in the order and copies only those present in `ALLOWED_FIELDS` into a new map. Fields like `customer_id`, `contains_hazmat`, `product_name`, `product_category`, `weight_kg`, and `shipping_zone` are silently dropped.
3. **Serialize and publish** — `marshal().json()` converts the filtered map back to JSON and the `to()` endpoint publishes it to the analytics topic.

The allowlist is defined as a `static final Set<String>`.
This makes it easy to review in code review and easy to test — the set is the single source of truth for what survives the filter.

### Spring Boot

```java
@Component
public class ContentFilterRoute extends RouteBuilder {

    private static final Set<String> ALLOWED_FIELDS = Set.of(
        "order_id", "item_sku", "quantity", "amount",
        "destination_country", "shipping_priority", "status"
    );

    @Override
    public void configure() {
        from("kafka:eip.orders.enriched?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=filter-demo")
            .routeId("content-filter")
            .unmarshal().json()
            .log("Filtering PII from order ${body[order_id]}")
            .process(exchange -> {
                var order = exchange.getIn().getBody(Map.class);
                var filtered = new LinkedHashMap<String, Object>();
                for (var entry : ((Map<String, Object>) order).entrySet()) {
                    if (ALLOWED_FIELDS.contains(entry.getKey())) {
                        filtered.put(entry.getKey(), entry.getValue());
                    }
                }
                exchange.getIn().setBody(filtered);
            })
            .log("Filtered: ${body}")
            .marshal().json()
            .to("kafka:eip.orders.analytics?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

`@Component` replaces `@ApplicationScoped` — the route logic is identical.

## Test templates

The test uses two JSON templates that make the filtering behavior visible at a glance.

The input template is the enriched order — a message with the full set of fields including product catalog data and customer references:

```json
{
  "order_id": "ORD-${id}",
  "customer_id": "CLIENT-${id}",
  "item_sku": "${sku}",
  "quantity": ${quantity},
  "amount": ${amount},
  "destination_country": "${country}",
  "contains_hazmat": ${hazardous},
  "status": "NEW",
  "product_name": "${productName}",
  "product_category": "${productCategory}",
  "weight_kg": "${weightKg}",
  "shipping_zone": "${shippingZone}"
}
```

This template contains twelve fields.
The content filter should strip five of them: `customer_id`, `contains_hazmat`, `product_name`, `product_category`, `weight_kg`, and `shipping_zone`.

The expected output template is the filtered order — only the fields allowed by the route's allowlist:

```json
{
  "order_id": "ORD-${id}",
  "item_sku": "${sku}",
  "quantity": ${quantity},
  "amount": ${amount},
  "destination_country": "${country}",
  "status": "NEW"
}
```

Six fields survive.
Placing these two templates side by side is the clearest possible documentation of what the filter does — no prose needed.

## Why template-based validation matters for content filters

A content filter's correctness has two dimensions:

1. **The right fields are present.** The output must contain `order_id`, `item_sku`, `quantity`, `amount`, `destination_country`, and `status`.
2. **The wrong fields are absent.** The output must *not* contain `customer_id`, `contains_hazmat`, `product_name`, `product_category`, `weight_kg`, or `shipping_zone`.

Field-by-field assertions naturally cover the first dimension.
But they are blind to the second.
A test that only asserts the six expected fields will pass even if `customer_id` leaks through — because it never checks for the absence of fields it does not expect.

Citrus's template-based validation covers both dimensions.
When the test receives a message and validates it against the filtered order template, it compares the entire JSON structure.
If the actual output contains a field that the template does not, the assertion fails.
This makes the template a structural contract: everything in the template must be present, and nothing outside the template is allowed.

For a content filter — especially one whose purpose is security or privacy — proving what is absent is more important than proving what is present.

## Test infrastructure

The Docker Compose stack includes Kafka and Redis (Redis is used by the content enricher in the same example project; the content filter test only needs Kafka).

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

The lifecycle pattern is the same as in the other transformation examples: Citrus's `testcontainers()` DSL brings up the Docker Compose stack, `waitFor().http()` blocks until the infrastructure is ready, and `afterSuite` tears everything down after the tests complete.

## The content filter test

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
    class ContentFilterTest {

        @Test
        public void shouldStripNonAllowedFieldsFromOrder() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-DEF-77")
                    .variable("quantity", 1)
                    .variable("amount", 89.99)
                    .variable("country", "GB")
                    .variable("hazardous", false)
                    .variable("productName", "Running Shoes")
                    .variable("productCategory", "Footwear")
                    .variable("weightKg", "1.2")
                    .variable("shippingZone", "ZONE-2")
            );

            t.given(waitForCamelRouteStarted("content-filter", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.enriched")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/enriched-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.analytics?consumerGroup=citrus-analytics-group")
                    .message()
                    .body(Resources.create("templates/filtered-order.json"))
            );
        }
    }
}
```

The test defines ten variables, but the filtered output template only uses six of them.
This asymmetry is the test's assertion in disguise.

**Given — set up variables and wait for the route.**

The first six variables (`id`, `sku`, `quantity`, `amount`, `country`, `hazardous`) describe the core order.
The last four (`productName`, `productCategory`, `weightKg`, `shippingZone`) are product enrichment fields that should *not* survive the filter.

These four variables are needed to populate the *input* template (the enriched order), but they should be absent from the *output* (the filtered order).
The filtered order template does not reference `${productName}`, `${productCategory}`, `${weightKg}`, or `${shippingZone}` at all.
It also does not reference `${hazardous}`, because `contains_hazmat` and `customer_id` are stripped by the allowlist.

**When — send the enriched order.**

The test sends a fully enriched order to `kafka:eip.orders.enriched` — all twelve fields are present in the message body.
This simulates the output of the content enricher route upstream in the pipeline.

**Then — receive and validate the filtered order.**

The test consumes from `kafka:eip.orders.analytics` and validates the body against the filtered order template.
Citrus compares the entire JSON structure: the six fields in the template must be present with the correct values, and no additional fields are allowed.

If the route's `ALLOWED_FIELDS` set accidentally included `customer_id`, the actual output would contain seven fields while the template expects six — the assertion would fail.
If a developer later added a new field to the enriched order (say, `supplier_cost`) without updating the filter's allowlist, the field would be automatically excluded by the allowlist approach and the test would continue to pass.
But if the same developer used a blocklist and forgot to add `supplier_cost` to the exclusion list, the field would leak through and the template assertion would catch it.

The `fork=true` option on the send operation avoids a blocking situation and handles the asynchronous processing:
the enricher needs time to consume the message, call the Redis lookup route, merge the results, and publish the enriched order.
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
    class ContentFilterTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldStripNonAllowedFieldsFromOrder() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-DEF-77")
                    .variable("quantity", 1)
                    .variable("amount", 89.99)
                    .variable("country", "GB")
                    .variable("hazardous", false)
                    .variable("productName", "Running Shoes")
                    .variable("productCategory", "Footwear")
                    .variable("weightKg", "1.2")
                    .variable("shippingZone", "ZONE-2")
            );

            t.given(waitForCamelRouteStarted("content-filter", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.enriched")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/enriched-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.analytics?consumerGroup=citrus-analytics-group")
                    .message()
                    .body(Resources.create("templates/filtered-order.json"))
            );
        }
    }
}
```

The test logic is identical.
The differences are confined to the framework annotations:

| Concern                | Quarkus                                | Spring Boot                                |
|------------------------|----------------------------------------|--------------------------------------------|
| Test bootstrap         | `@QuarkusTest`                         | `@SpringBootTest` + `@CamelSpringBootTest` |
| Citrus integration     | `@CitrusSupport`                       | `@CitrusSpringSupport`                     |
| CamelContext injection | `@Inject` + `@BindToRegistry`          | `@Autowired`                               |
| TestCaseRunner scope   | Class-level field                      | Nested class field with `@CitrusResource`  |
| Infrastructure config  | `@CitrusConfiguration` auto-discovered | `@ContextConfiguration` explicit           |

## The content filter in the transformation pipeline

The content filter does not operate in isolation.
In the example project, it is part of a three-stage transformation pipeline:

1. **Message Translator** — Reads orders from `eip.orders.external` in the partner's format, translates them to the canonical format, and publishes to `eip.orders.placed`.
2. **Content Enricher** — Reads from `eip.orders.placed`, looks up product data from Redis, merges the fields, and publishes to `eip.orders.enriched`.
3. **Content Filter** — Reads from `eip.orders.enriched`, strips non-allowed fields, and publishes to `eip.orders.analytics`.

Each test in the suite is independent — it sends directly to the route's input topic and receives from the route's output topic.
But the templates form a chain: the enriched order template that the content filter test uses as *input* is the same template that the content enricher test uses as *expected output*.
This consistency ensures that the templates accurately represent the data flowing between stages.

Testing each stage independently rather than testing the full pipeline end-to-end has a practical advantage: when a test fails, you know exactly which stage broke.
If the content filter test fails, the problem is in the filter's allowlist or its processing logic — not in the translator or the enricher upstream.

## Evolving the allowlist safely

Content filters have a maintenance risk: the allowlist can become stale when the upstream schema evolves.
If a new field is added to the enriched order (say, `estimated_delivery`), the filter will automatically exclude it — which is the safe default for a security-oriented allowlist, but might not be the desired behavior for the analytics consumer.

The integration test acts as a safety net for this evolution.
When you add `estimated_delivery` to the enriched order schema:

- If the analytics consumer *should* receive it, update the allowlist in the route and add the field to the filtered order template. The test will fail until both are updated, ensuring they stay in sync.
- If the analytics consumer *should not* receive it, do nothing. The allowlist excludes it automatically, and the template-based assertion confirms the field is absent.

This is the advantage of an allowlist over a blocklist: new fields default to *excluded*, and you must explicitly opt in.
With a blocklist, new fields default to *included*, and you must remember to add them to the exclusion list — a task that is easy to forget and whose failure is silent until someone audits the data flow.

## Key takeaways

- **Template comparison validates both presence and absence.** A filtered order template that contains six fields implicitly asserts that the other six fields from the input are stripped. Field-by-field assertions cannot make this guarantee.
- **Allowlists are safer than blocklists for security-sensitive filters.** New fields are automatically excluded by an allowlist. A blocklist requires manual updates whenever the upstream schema adds a field — a maintenance burden that is easy to overlook.
- **Input variables that don't appear in the output template are the assertion.** The test defines `${productName}`, `${productCategory}`, `${weightKg}`, and `${shippingZone}` to populate the input, but the output template never references them. Their absence from the output template is what proves the filter works.
- **Independent stage tests pinpoint failures.** Testing the content filter in isolation — sending to its input topic and receiving from its output topic — means a failure points directly to the filter, not to the translator or enricher upstream.
- **Two runtimes, one test pattern.** The test logic is identical across Quarkus and Spring Boot. The allowlist, the templates, and the given-when-then structure are runtime-agnostic — only the framework annotations differ.
