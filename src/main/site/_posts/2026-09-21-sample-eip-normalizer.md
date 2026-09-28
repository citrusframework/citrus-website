---
layout: sample
title: Testing the Normalizer Pattern with Citrus
name: normalizer
image: /img/icons/camel.png
folder: examples/13-aggregator
group: eip
description: Testing the Normalizer EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/normalizer/
---

Integration projects rarely have the luxury of a single data format.
A shipping platform receives orders from a web application that sends JSON with camelCase field names, a mobile app that uses its own JSON schema, a legacy partner system that sends CSV, and a B2B EDI gateway that delivers XML.
Every source represents the same business concept — an order — but each uses different field names, different casing conventions, and sometimes different serialization formats entirely.

Without normalization, the downstream processing logic must understand every variant.
Each new partner or source format means another branch in the consumer code, another set of field mappings, another opportunity for a missed edge case.
Over time, this fan-out of format awareness turns the consumer into a brittle monolith that knows far too much about its upstream producers.

The [Normalizer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Normalizer.html) pattern, described by Hohpe and Woolf, solves this by placing a translation layer between the raw input and the business logic.
Each incoming format is detected and translated into a single *canonical form* before it reaches any downstream processing.
After normalization, every consumer sees the same field names, the same structure, and the same data types — regardless of where the message originated.

In practice, a normalizer is a composed pattern: it combines a content-based router (to detect the source format) with message translators (to convert each format to canonical).
[Apache Camel](https://camel.apache.org) does not have a dedicated `normalize()` EIP — instead, you build a normalizer from existing building blocks: per-source routes, inline processors for field mapping, and a shared output topic that carries the canonical schema.

Testing a normalizer is about coverage and correctness.
Each partner format is a separate translation path, and every path must be verified independently.
The test must prove that Partner A's `orderId` becomes `order_id`, that Partner B's `buyer_ref` becomes `customer_id`, and that Partner C's `item_code` becomes `item_sku` — all landing on the same output topic in the same canonical shape.
This is exactly where [Citrus](https://citrusframework.org) shines: you can send partner-specific input, receive the normalized output, and validate field-by-field that the translation is correct.

## The scenario

Three partners send orders to a shared platform, each using a different naming convention:

| Partner   | Order ID field | Customer field | SKU field   | Quantity field | Amount field |
|-----------|----------------|----------------|-------------|----------------|--------------|
| Partner A | `orderId`      | `client`       | `product`   | `count`        | `total`      |
| Partner B | `order_number` | `buyer_ref`    | `sku`       | `qty`          | `price`      |
| Partner C | `po_id`        | `account`      | `item_code` | `units`        | `value`      |
| Canonical | `order_id`     | `customer_id`  | `item_sku`  | `quantity`     | `amount`     |

Each partner has its own Kafka input topic (`eip.orders.partner-a`, `eip.orders.partner-b`, `eip.orders.partner-c`).
A dedicated Camel route per partner consumes from the partner's topic, translates the fields to the canonical schema, and publishes to a shared `eip.orders.normalized` topic.

The canonical output also adds two metadata fields: `source` (which partner sent it) and `status` (always `"NEW"` for freshly normalized orders).

## The Camel routes

### Quarkus

```java
@ApplicationScoped
public class NormalizerRoute extends RouteBuilder {

    @Override
    public void configure() {
        // Partner A: uses "orderId", "client", "product", "count", "total"
        from("kafka:eip.orders.partner-a?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=normalizer-demo")
            .routeId("normalizer-partner-a")
            .unmarshal().json()
            .log("Partner A order received: ${body}")
            .process(exchange -> {
                var src = exchange.getIn().getBody(Map.class);
                var canonical = new LinkedHashMap<String, Object>();
                canonical.put("order_id", src.get("orderId"));
                canonical.put("customer_id", src.get("client"));
                canonical.put("item_sku", src.get("product"));
                canonical.put("quantity", src.get("count"));
                canonical.put("amount", src.get("total"));
                canonical.put("source", "PARTNER_A");
                canonical.put("status", "NEW");
                exchange.getIn().setBody(canonical);
            })
            .marshal().json()
            .to("kafka:eip.orders.normalized?brokers={% raw %}{{kafka.brokers}}{% endraw %}");

        // Partner B: uses "order_number", "buyer_ref", "sku", "qty", "price"
        from("kafka:eip.orders.partner-b?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=normalizer-demo")
            .routeId("normalizer-partner-b")
            .unmarshal().json()
            .log("Partner B order received: ${body}")
            .process(exchange -> {
                var src = exchange.getIn().getBody(Map.class);
                var canonical = new LinkedHashMap<String, Object>();
                canonical.put("order_id", src.get("order_number"));
                canonical.put("customer_id", src.get("buyer_ref"));
                canonical.put("item_sku", src.get("sku"));
                canonical.put("quantity", src.get("qty"));
                canonical.put("amount", src.get("price"));
                canonical.put("source", "PARTNER_B");
                canonical.put("status", "NEW");
                exchange.getIn().setBody(canonical);
            })
            .marshal().json()
            .to("kafka:eip.orders.normalized?brokers={% raw %}{{kafka.brokers}}{% endraw %}");

        // Partner C: uses "po_id", "account", "item_code", "units", "value"
        from("kafka:eip.orders.partner-c?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=normalizer-demo")
            .routeId("normalizer-partner-c")
            .unmarshal().json()
            .log("Partner C order received: ${body}")
            .process(exchange -> {
                var src = exchange.getIn().getBody(Map.class);
                var canonical = new LinkedHashMap<String, Object>();
                canonical.put("order_id", src.get("po_id"));
                canonical.put("customer_id", src.get("account"));
                canonical.put("item_sku", src.get("item_code"));
                canonical.put("quantity", src.get("units"));
                canonical.put("amount", src.get("value"));
                canonical.put("source", "PARTNER_C");
                canonical.put("status", "NEW");
                exchange.getIn().setBody(canonical);
            })
            .marshal().json()
            .to("kafka:eip.orders.normalized?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }
}
```

Each partner gets its own route with a dedicated `routeId`.
The inline `process()` block performs the field translation — mapping partner-specific field names to the canonical schema.
There is no shared router or choice block: each partner has its own Kafka topic, so format detection happens implicitly through topic assignment.

This design is intentional.
A single route with a `choice()` block that inspects a header or content to determine the partner would centralize the normalizer logic, but it would also create a coupling point: adding a new partner means modifying the existing route.
Separate routes per partner are independently deployable, independently testable, and independently scalable.

### Spring Boot

The Spring Boot variant replaces `@ApplicationScoped` with `@Component` — the route logic is identical:

```java
@Component
public class NormalizerRoute extends RouteBuilder {

    @Override
    public void configure() {
        // Partner A route — same logic as Quarkus
        from("kafka:eip.orders.partner-a?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=normalizer-demo")
            .routeId("normalizer-partner-a")
            .unmarshal().json()
            .process(exchange -> {
                var src = exchange.getIn().getBody(Map.class);
                var canonical = new LinkedHashMap<String, Object>();
                canonical.put("order_id", src.get("orderId"));
                canonical.put("customer_id", src.get("client"));
                canonical.put("item_sku", src.get("product"));
                canonical.put("quantity", src.get("count"));
                canonical.put("amount", src.get("total"));
                canonical.put("source", "PARTNER_A");
                canonical.put("status", "NEW");
                exchange.getIn().setBody(canonical);
            })
            .marshal().json()
            .to("kafka:eip.orders.normalized?brokers={% raw %}{{kafka.brokers}}{% endraw %}");

        // Partner B and C routes follow the same pattern ...
    }
}
```

## Why normalizer tests need per-partner isolation

Testing a normalizer is not about testing one route — it is about testing N independent translation paths that must all produce output in the same canonical shape.
This creates a specific challenge: all three partner routes write to the same `eip.orders.normalized` topic.
If the tests are not carefully isolated, Partner A's test could accidentally consume Partner B's normalized output and pass when it should have failed.

The solution is simple: each test uses a dedicated Kafka consumer group.
Partner A's test consumes from `citrus-normalized-a-group`, Partner B from `citrus-normalized-b-group`, and Partner C from `citrus-normalized-c-group`.
Please keep in mind that the setup of Kafka consumer groups always goes hand in hand with the offset-reset setting (`earliest`, `latest`) that is being used.
Because Kafka tracks offsets per consumer group independently, each test sees only the messages it produces — no cross-contamination.

## Message templates

The tests use JSON templates with Citrus variable placeholders.
Each partner has its own input template reflecting that partner's field naming convention, but all three share a single output template representing the canonical schema.

### Partner A input

```json
{
  "orderId": "ORD-${id}",
  "client": "CUST-${id}",
  "product": "${sku}",
  "count": ${quantity},
  "total": ${amount}
}
```

### Partner B input

```json
{
  "order_number": "ORD-${id}",
  "buyer_ref": "CUST-${id}",
  "sku": "${sku}",
  "qty": ${quantity},
  "price": ${amount}
}
```

### Partner C input

```json
{
  "po_id": "ORD-${id}",
  "account": "CUST-${id}",
  "item_code": "${sku}",
  "units": ${quantity},
  "value": ${amount}
}
```

### Canonical output (shared)

```json
{
  "order_id": "ORD-${id}",
  "customer_id": "CUST-${id}",
  "item_sku": "${sku}",
  "quantity": ${quantity},
  "amount": ${amount},
  "source": "${source}",
  "status": "NEW"
}
```

Notice the symmetry: every partner input template uses different field names for the same business concepts, but all three are expected to produce output matching the same canonical template.
The `${source}` variable changes per test (`PARTNER_A`, `PARTNER_B`, `PARTNER_C`), which verifies that the normalizer correctly tags the origin.

This template-based approach is a key Citrus technique.
Rather than constructing JSON strings in Java code or using assertion libraries to pick apart individual fields, you declare the full expected message shape as a resource file.
Citrus performs a structural comparison — every field in the template must match the received message, and unexpected fields cause a validation failure.
The important detail here is that Citrus is able to ignore the ordering of fields for Json message payloads when validating received messages with the use of templates.
The result is a test that reads like a specification: "given this input shape, I expect this output shape."

## The normalizer tests

Each partner gets its own nested test class.
The structure is identical across all three: set up variables, wait for the route, send a partner-specific message, receive and validate the canonical output.

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
    class NormalizerPartnerATest {

        @Test
        public void shouldNormalizePartnerAOrderToCanonicalFormat() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-PA-01")
                    .variable("quantity", 5)
                    .variable("amount", 149.95)
                    .variable("source", "PARTNER_A")
            );

            t.given(waitForCamelRouteStarted("normalizer-partner-a", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.partner-a")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/partner-a-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "ORD-${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.normalized?consumerGroup=citrus-normalized-a-group")
                    .message()
                    .body(Resources.create("templates/normalized-order.json"))
            );
        }
    }

    @Nested
    class NormalizerPartnerBTest {

        @Test
        public void shouldNormalizePartnerBOrderToCanonicalFormat() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-PB-02")
                    .variable("quantity", 3)
                    .variable("amount", 89.97)
                    .variable("source", "PARTNER_B")
            );

            t.given(waitForCamelRouteStarted("normalizer-partner-b", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.partner-b")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/partner-b-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "ORD-${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.normalized?consumerGroup=citrus-normalized-b-group")
                    .message()
                    .body(Resources.create("templates/normalized-order.json"))
            );
        }
    }

    @Nested
    class NormalizerPartnerCTest {

        @Test
        public void shouldNormalizePartnerCOrderToCanonicalFormat() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-PC-03")
                    .variable("quantity", 10)
                    .variable("amount", 199.90)
                    .variable("source", "PARTNER_C")
            );

            t.given(waitForCamelRouteStarted("normalizer-partner-c", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.partner-c")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/partner-c-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "ORD-${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.normalized?consumerGroup=citrus-normalized-c-group")
                    .message()
                    .body(Resources.create("templates/normalized-order.json"))
            );
        }
    }
}
```

Let's walk through the test for Partner A in detail — the other two follow the same structure.

**Given — set up variables and wait for the route.**
The test generates a random 4-digit ID and defines the order attributes.
The `source` variable is set to `"PARTNER_A"` — this will be used to validate the `source` field in the canonical output.
Then it waits for the `normalizer-partner-a` route to reach `Started` status via the Control Bus utility.

**When — send a partner-specific message.**
A single message goes to the `eip.orders.partner-a` Kafka topic.
The message body is loaded from the `partner-a-order.json` template, which uses Partner A's field names (`orderId`, `client`, `product`, `count`, `total`).
The Citrus variable placeholders (`${id}`, `${sku}`, `${quantity}`, `${amount}`) are resolved at runtime, so each test run uses unique values.

**Then — verify the canonical output.**
The test receives a message from the `eip.orders.normalized` topic using a dedicated consumer group (`citrus-normalized-a-group`).
The received message is validated against the `normalized-order.json` template, which uses the canonical field names (`order_id`, `customer_id`, `item_sku`, `quantity`, `amount`).
Because the same Citrus variables power both the input and output templates, the test proves end-to-end that `orderId` → `order_id`, `client` → `customer_id`, and so on.

### Spring Boot test

```java
@SpringBootTest(classes = AggregatorApplication.class)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class NormalizerPartnerATest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldNormalizePartnerAOrderToCanonicalFormat() {
            // Test logic is identical to the Quarkus variant
            // ...
        }
    }

    // NormalizerPartnerBTest and NormalizerPartnerCTest follow the same pattern
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

## What the tests actually prove

It is worth stepping back to consider what these normalizer tests verify — and what they do not.

**Field mapping correctness.**
Each test sends a message with partner-specific field names and validates that the output contains the canonical field names with the correct values.
If a developer swaps `src.get("client")` with `src.get("customer")` in Partner A's translator, the test fails because the canonical output will have a `null` value for `customer_id` instead of the expected `CUST-${id}`.

**Source tagging.**
The `source` field in the canonical output identifies which partner originated the order.
Each test sets the `${source}` variable to the expected value and validates it in the output template.
If Partner B's route mistakenly tags the source as `"PARTNER_A"`, the test catches it.

**Schema consistency.**
All three tests validate against the *same* canonical output template.
This means the tests implicitly enforce that all three translation paths produce output in the same shape.
If Partner C's translator forgets to include the `status` field, that test fails — even though the other two partners include it correctly.

**What else could be added:** Right now the normalizer tests focus on the golden path: well-formed partner input in, correctly translated canonical output out.
Further separate negative-path tests would cover the behavior when a partner sends malformed input (missing fields, wrong types, extra fields).

## Scaling normalizers: the N-partner problem

The example has three partners.
In a real system, the number of partners tends to grow.
Each new partner means a new Kafka topic, a new Camel route, a new input template, and a new test class.
The test structure scales linearly — adding Partner D requires:

1. A new `normalizer-partner-d` route with the field mapping.
2. A new `partner-d-order.json` input template.
3. A new `NormalizerPartnerDTest` nested class that follows the same pattern.
4. The existing `normalized-order.json` output template is reused unchanged.

What if the number of partners grows significantly? 
The risk arises that the normalizer routes and the tests themselves become a maintenance burden.

The Citrus tests can be optimized to take care of this risk.
Adding a new partner does not change the expected output shape — it only adds a new input path that must produce the same shape.
Step 4 is the key insight: the canonical output template is shared.

This means we can have one single parameterized test that has the input Kafka topic name (where Citrus sends its messages to) and the partner template name (`partner-X-order.json`) as parameters.
Adding new partners to the test coverage then resides to just creating the partner template input file together with the input Kafka topic name.
The test validate the contract, not the implementation.
This is the normalizer pattern delivering on its promise: N inputs, one canonical output.

For the test infrastructure setup, shared test utilities, runtime wiring, dependencies, and how to run the tests, see the [Camel EIP examples](/samples/camel-eip/) overview page.

## Key takeaways

- **Each translation path needs its own test.** A normalizer with three partners has three independent code paths. Testing only one partner and assuming the others work the same way misses mapping bugs in the untested paths.
- **A shared canonical output template enforces consistency.** Using the same expected-output template across all partner tests guarantees that every translation path produces the same canonical shape.
- **Consumer group isolation prevents cross-test interference.** When multiple normalizer routes write to the same output topic, each test must use its own Kafka consumer group to avoid consuming another test's messages.
- **Citrus variables link input to output.** Setting `${id}`, `${sku}`, `${amount}` once and using them in both the input and output templates creates an end-to-end assertion chain — the test proves that specific values survive the translation intact.
- **Normalizer tests are fast.** Unlike aggregator or resequencer tests that must wait for completion conditions or batch timeouts, normalizer tests verify a stateless 1:1 translation. Output appears almost immediately, so retry windows can be shorter.
- **`fork=true` on the `send` handles multi-hop async processing.** The translator processes messages asynchronously. Rather than inserting fixed sleeps, Citrus initializes the `receive` immediately until the translated message appears. The non-blocking send operation gives the full pipeline time to complete and the Kafka consumer time to initialize without any racing conditions.
- **Two runtimes, one test pattern.** Whether you run on Quarkus or Spring Boot, the test structure — send partner-specific input, receive canonical output, validate the mapping — stays the same. Only the bootstrap annotations and dependency injection change.
