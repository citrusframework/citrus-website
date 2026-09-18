---
layout: sample
title: Testing the Content Enricher Pattern with Citrus
name: content-enricher
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Content Enricher EIP in Apache Camel with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/content-enricher/
---

A message usually carries all the data that the sender knows.
But the receiving service may require even more data to properly process the message.

Imagine an order event that arrives on a Kafka topic. 
The order carries a product SKU, a quantity, and a total amount.
That is everything the order service as a sender knows.
The downstream shipping service, however, needs the product's weight, its shipping zone, and its category to calculate delivery costs and select a carrier.
The order service does not have this data — it lives in a product catalog backed by Redis.

You could make the order service look up all required product data before publishing the event.
But that couples the order service to the product catalog, adds latency to the order flow, and means every new consumer with different data needs forces a change to the producer.
The better approach is to let the consumer enrich the message itself, pulling in the additional data from the source that has it.

The [Content Enricher](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DataEnricher.html) pattern, described by Hohpe and Woolf, solves exactly this problem.
It receives a message, looks up supplementary data from an external source — a database, a cache, an API — and merges the result into the original message before passing it along.

[Apache Camel](https://camel.apache.org) implements this with the `enrich()` EIP, which calls an external resource and combines the response with the original message using an `AggregationStrategy`.
The original message and the enrichment response meet inside the strategy, where you decide which fields to keep, add, or overwrite.

Testing a content enricher is more involved than testing a stateless translator or router.
The enricher depends on an external data source — in our case, Redis — which means the test must ensure that the data source contains the expected data before sending any messages.
An integration test with [Citrus](https://citrusframework.org) proves that the entire pipeline works: Kafka consumption, Redis lookup, field merging, and Kafka production — all against real infrastructure.

## The scenario

Orders arrive on the `eip.orders.placed` topic in a canonical format: `order_id`, `customer_id`, `item_sku`, `quantity`, `amount`, and a few other fields.
The content enricher route reads each order, looks up the product's details from a Redis-backed catalog using the `item_sku`, merges four additional fields into the order (`product_name`, `product_category`, `weight_kg`, `shipping_zone`), and publishes the enriched order to `eip.orders.enriched`.

## The product catalog

Before looking at the route, it is worth understanding the data source.
The `RedisProductCatalog` is a simple service that stores product metadata in Redis hashes and provides a `lookup(sku)` method.

### Quarkus

```java
@ApplicationScoped
@Startup
public class RedisProductCatalog {

    private final HashCommands<String, String, String> hash;

    @Inject
    public RedisProductCatalog(RedisDataSource ds) {
        this.hash = ds.hash(String.class, String.class, String.class);
        seedCatalog();
    }

    private void seedCatalog() {
        addProduct("SKU-ABC-42", "Wireless Headphones", "29.99",
                   "Electronics", "0.3", "ZONE-1");
        addProduct("SKU-DEF-77", "Running Shoes", "89.99",
                   "Footwear", "1.2", "ZONE-2");
        addProduct("SKU-GHI-13", "Coffee Maker", "149.99",
                   "Appliances", "4.5", "ZONE-3");
    }

    public Map<String, String> lookup(String sku) {
        Map<String, String> product = hash.hgetall("product:" + sku);
        if (product == null || product.isEmpty()) {
            return Map.of(
                "name", "Unknown Product",
                "category", "General",
                "weight_kg", "1.0",
                "shipping_zone", "ZONE-1"
            );
        }
        return product;
    }
}
```

### Spring Boot

```java
@Component
public class RedisProductCatalog {

    private final HashOperations<String, String, String> hash;

    @Value("${redis.catalog.seed:true}")
    boolean seedEnabled;

    public RedisProductCatalog(StringRedisTemplate redisTemplate) {
        this.hash = redisTemplate.opsForHash();
    }

    @PostConstruct
    void init() {
        if (seedEnabled) seedCatalog();
    }

    public void seedCatalog() {
        addProduct("SKU-ABC-42", "Wireless Headphones", "29.99",
                   "Electronics", "0.3", "ZONE-1");
        addProduct("SKU-DEF-77", "Running Shoes", "89.99",
                   "Footwear", "1.2", "ZONE-2");
        addProduct("SKU-GHI-13", "Coffee Maker", "149.99",
                   "Appliances", "4.5", "ZONE-3");
    }

    public Map<String, String> lookup(String sku) {
        Map<String, String> product = hash.entries("product:" + sku);
        if (product == null || product.isEmpty()) {
            return Map.of(
                "name", "Unknown Product",
                "category", "General",
                "weight_kg", "1.0",
                "shipping_zone", "ZONE-1"
            );
        }
        return product;
    }
}
```

Both implementations seed the catalog with known products at startup and provide a fallback for unknown SKUs.
The Quarkus variant uses Quarkus Redis extensions (`RedisDataSource`), while Spring Boot uses `StringRedisTemplate` — the lookup logic is the same.

Notice the fallback in `lookup()`: when a SKU is not found, the method returns a default product rather than null or an exception.
This is a deliberate design choice that affects testing — it means the enricher will always produce a complete output, even for unknown SKUs.
A test could verify this fallback behavior by sending an order with an unrecognized SKU and asserting that the enriched fields contain the defaults.

## The Camel route

### Quarkus

```java
@ApplicationScoped
public class ContentEnricherRoute extends RouteBuilder {

    @Inject
    RedisProductCatalog productCatalog;

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=enricher-demo")
            .routeId("content-enricher")
            .unmarshal().json()
            .log("Enriching order ${body[order_id]}")
            .enrich("direct:redis-product-lookup", (oldExchange, newExchange) -> {
                var order = oldExchange.getIn().getBody(Map.class);
                var product = newExchange.getIn().getBody(Map.class);
                var enriched = new LinkedHashMap<>(order);
                enriched.put("product_name", product.get("name"));
                enriched.put("product_category", product.get("category"));
                enriched.put("weight_kg", product.get("weight_kg"));
                enriched.put("shipping_zone", product.get("shipping_zone"));
                oldExchange.getIn().setBody(enriched);
                return oldExchange;
            })
            .log("Enriched from Redis: ${body[item_sku]} → ${body[product_name]} (${body[shipping_zone]})")
            .marshal().json()
            .to("kafka:eip.orders.enriched?brokers={% raw %}{{kafka.brokers}}{% endraw %}");

        from("direct:redis-product-lookup")
            .routeId("redis-product-lookup")
            .process(exchange -> {
                var order = exchange.getIn().getBody(Map.class);
                String sku = (String) order.get("item_sku");
                Map<String, String> product = productCatalog.lookup(sku);
                exchange.getIn().setBody(product);
            });
    }
}
```

The route defines two interconnected pieces:

**The main route** (`content-enricher`) reads from the Kafka input topic, deserializes the JSON order, and calls `enrich("direct:redis-product-lookup", ...)`.
The `enrich()` EIP sends a copy of the current exchange to the specified endpoint and then invokes the `AggregationStrategy` with two exchanges: the original message (`oldExchange`) and the enrichment result (`newExchange`).
Inside the strategy, the code creates a new map that contains all the original order fields plus the four product fields from Redis.

**The lookup route** (`redis-product-lookup`) is a `direct:` route that extracts the SKU from the order body and calls `productCatalog.lookup(sku)`.
Separating the lookup into its own route keeps the main route focused on the enrichment logic and makes the lookup independently testable.

### Spring Boot

The Spring Boot variant is identical except for the annotations:

```java
@Component
public class ContentEnricherRoute extends RouteBuilder {

    @Autowired
    RedisProductCatalog productCatalog;

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=enricher-demo")
            .routeId("content-enricher")
            .unmarshal().json()
            .log("Enriching order ${body[order_id]}")
            .enrich("direct:redis-product-lookup", (oldExchange, newExchange) -> {
                var order = oldExchange.getIn().getBody(Map.class);
                var product = newExchange.getIn().getBody(Map.class);
                var enriched = new LinkedHashMap<>(order);
                enriched.put("product_name", product.get("name"));
                enriched.put("product_category", product.get("category"));
                enriched.put("weight_kg", product.get("weight_kg"));
                enriched.put("shipping_zone", product.get("shipping_zone"));
                oldExchange.getIn().setBody(enriched);
                return oldExchange;
            })
            .log("Enriched from Redis: ${body[item_sku]} → ${body[product_name]} (${body[shipping_zone]})")
            .marshal().json()
            .to("kafka:eip.orders.enriched?brokers={% raw %}{{kafka.brokers}}{% endraw %}");

        from("direct:redis-product-lookup")
            .routeId("redis-product-lookup")
            .process(exchange -> {
                var order = exchange.getIn().getBody(Map.class);
                String sku = (String) order.get("item_sku");
                Map<String, String> product = productCatalog.lookup(sku);
                exchange.getIn().setBody(product);
            });
    }
}
```

`@Component` and `@Autowired` replace `@ApplicationScoped` and `@Inject` — the route logic stays the same.

## Why enricher tests need real infrastructure

A content enricher has a dependency that most routing patterns do not: an external data source.
This creates a testing challenge.

You could mock the Redis lookup and test the `AggregationStrategy` in isolation.
That would verify the merging logic — the four `put()` calls that add product fields to the order.
But it would not verify that the `productCatalog.lookup()` method actually retrieves the correct data from Redis, that the Redis connection is configured properly, or that the seeded product data matches what the test expects.

An integration test with real Kafka and real Redis catches issues at every layer: Kafka deserialization, the `enrich()` EIP invocation, the `direct:` route dispatch, the Redis hash lookup, the aggregation strategy, and Kafka serialization of the enriched result.

## Test templates

The test uses two JSON templates.
The input template represents a canonical order (the format produced by the message translator):

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

The expected output template adds the four enrichment fields:

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

The enrichment fields use their own Citrus variables (`${productName}`, `${productCategory}`, `${weightKg}`, `${shippingZone}`).
These values must match what is seeded in the Redis product catalog for the given SKU.
This tight coupling between the test variables and the seeded data is intentional — it is the assertion that the enricher looked up the right product and merged the right fields.

## Test infrastructure

The Docker Compose stack for this example includes both Kafka and Redis:

```yaml
services:
  kafka:
    image: docker.io/apache/kafka:4.3.1
    ports:
      - "9092:9092"
      - "9094:9094"
    # ... Kafka configuration ...

  kafka-ui:
    image: docker.io/provectuslabs/kafka-ui:v0.7.2
    ports:
      - "8090:8080"
    depends_on:
      kafka:
        condition: service_healthy

  redis:
    image: docker.io/library/redis:8.10.1-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
```

Redis is the enrichment source.
The `BeforeSuite` brings up the full compose stack; the `AfterSuite` tears it down.

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

The `waitFor().http().url("http://localhost:8090")` call blocks until the Kafka UI is reachable — a reliable proxy for the Kafka broker and Redis both being ready, since the UI depends on Kafka and both containers are part of the same compose stack.

## The content enricher test

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
    class ContentEnricherTest {

        @Test
        public void shouldEnrichOrderWithProductCatalogData() {
            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-ABC-42")
                    .variable("quantity", 3)
                    .variable("amount", 89.97)
                    .variable("country", "DE")
                    .variable("hazardous", false)
                    .variable("productName", "Wireless Headphones")
                    .variable("productCategory", "Electronics")
                    .variable("weightKg", "0.3")
                    .variable("shippingZone", "ZONE-1")
            );

            t.given(waitForCamelRouteStarted("content-enricher", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.placed")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/canonical-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.enriched?consumerGroup=citrus-enriched-group")
                    .message()
                    .body(Resources.create("templates/enriched-order.json"))
            );
        }
    }
}
```

Let's walk through this test carefully, because the variable setup reveals how the enrichment assertion works.

**Given — set up variables and wait for the route.**

The test defines ten variables split into two logical groups.
The first six (`id`, `sku`, `quantity`, `amount`, `country`, `hazardous`) populate the input template — they describe the order as it arrives on the Kafka topic.
The last four (`productName`, `productCategory`, `weightKg`, `shippingZone`) populate the *expected enrichment fields* in the output template.

These enrichment variables are set to values that match what the `RedisProductCatalog` seeds for SKU `SKU-ABC-42`: "Wireless Headphones", "Electronics", "0.3", "ZONE-1".
If the Redis catalog contained different data for this SKU, or if the enricher looked up the wrong SKU, or if the aggregation strategy mapped the fields incorrectly, the receive assertion would fail.

This is the critical design decision in the test: the enrichment variables are *not* arbitrary — they are the expected result of the Redis lookup for the given SKU.
The test author must know what data the catalog contains for `SKU-ABC-42` and set the variables accordingly.

**When — send the canonical order.**

A single message goes to `kafka:eip.orders.placed` using the canonical order template.
The body contains the six order fields; the four enrichment fields are not yet present.

**Then — receive and validate the enriched order.**

The test consumes from `kafka:eip.orders.enriched` and validates the body against the enriched order template.
This template contains all ten fields — the original six plus the four enrichment fields.
Citrus resolves all variables and performs a structural comparison.

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

    @Autowired
    RedisProductCatalog redisProductCatalog;

    @Nested
    class ContentEnricherTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        public void shouldEnrichOrderWithProductCatalogData() {
            redisProductCatalog.seedCatalog();

            t.given(
                createVariables()
                    .variable("id", "citrus:randomNumber(4)")
                    .variable("sku", "SKU-ABC-42")
                    .variable("quantity", 3)
                    .variable("amount", 89.97)
                    .variable("country", "DE")
                    .variable("hazardous", false)
                    .variable("productName", "Wireless Headphones")
                    .variable("productCategory", "Electronics")
                    .variable("weightKg", "0.3")
                    .variable("shippingZone", "ZONE-1")
            );

            t.given(waitForCamelRouteStarted("content-enricher", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.placed")
                    .message()
                    .fork(true)
                    .body(Resources.create("templates/canonical-order.json"))
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "${id}")
            );

            t.then(
                receive()
                    .endpoint("kafka:eip.orders.enriched?consumerGroup=citrus-enriched-group")
                    .message()
                    .body(Resources.create("templates/enriched-order.json"))
            );
        }
    }
}
```

The Spring Boot test has one notable addition: the explicit `redisProductCatalog.seedCatalog()` call at the start of the test method.

In the Quarkus variant, `RedisProductCatalog` is annotated with `@Startup` and seeds the catalog in its constructor — the data is always present by the time the test runs.
In Spring Boot, the `@PostConstruct` method checks a configuration flag (`redis.catalog.seed`), and the test explicitly calls `seedCatalog()` to guarantee the data is present regardless of configuration.
This is a small but important difference: it ensures the test is self-contained and does not depend on application startup order.

The rest of the test is identical — same variables, same templates, same given-when-then structure.

| Concern                | Quarkus                       | Spring Boot                                |
|------------------------|-------------------------------|--------------------------------------------|
| Test bootstrap         | `@QuarkusTest`                | `@SpringBootTest` + `@CamelSpringBootTest` |
| Citrus integration     | `@CitrusSupport`              | `@CitrusSpringSupport`                     |
| CamelContext injection | `@Inject` + `@BindToRegistry` | `@Autowired`                               |
| TestCaseRunner scope   | Class-level field             | Nested class field with `@CitrusResource`  |
| Redis seeding          | Automatic via `@Startup`      | Explicit `seedCatalog()` call              |

## The aggregation strategy under the lens

The `AggregationStrategy` inside the `enrich()` call is the heart of the content enricher pattern.
It is worth understanding what the test actually asserts about it.

```java
.enrich("direct:redis-product-lookup", (oldExchange, newExchange) -> {
    var order = oldExchange.getIn().getBody(Map.class);
    var product = newExchange.getIn().getBody(Map.class);
    var enriched = new LinkedHashMap<>(order);
    enriched.put("product_name", product.get("name"));
    enriched.put("product_category", product.get("category"));
    enriched.put("weight_kg", product.get("weight_kg"));
    enriched.put("shipping_zone", product.get("shipping_zone"));
    oldExchange.getIn().setBody(enriched);
    return oldExchange;
})
```

The strategy creates a *new* map from the original order (preserving all existing fields) and adds four fields from the Redis lookup result.
The test validates every aspect of this logic:

- **All original fields are preserved.** The enriched template contains `order_id`, `customer_id`, `item_sku`, `quantity`, `amount`, `destination_country`, `contains_hazmat`, and `status` — all carried forward from the input. If `new LinkedHashMap<>(order)` were accidentally replaced with `new LinkedHashMap<>()`, the receive assertion would fail because the original fields would be missing.
- **The correct product fields are added.** The template expects `product_name`, `product_category`, `weight_kg`, and `shipping_zone` with values matching the Redis seed data. If the strategy mapped `product.get("name")` to a wrong key (say, `"product_title"` instead of `"product_name"`), the assertion would catch it.
- **The lookup uses the right key.** The Redis lookup extracts `item_sku` from the order body. If it used the wrong field (say, `order_id`), the lookup would return the fallback defaults ("Unknown Product", "General"), and the enrichment variables would not match.

## What the test proves

This single test exercises a pipeline that spans two Camel routes, two infrastructure services, and a CDI/Spring-managed bean:

1. **Kafka consumer** — reads the order from `eip.orders.placed`.
2. **JSON deserialization** — `unmarshal().json()` converts bytes to a map.
3. **Enrich EIP dispatch** — the `enrich()` call sends the exchange to `direct:redis-product-lookup`.
4. **Redis lookup** — `productCatalog.lookup(sku)` queries Redis using the SKU as the key.
5. **Aggregation** — the strategy merges the product fields into the order map.
6. **JSON serialization** — `marshal().json()` converts the enriched map back to bytes.
7. **Kafka producer** — writes the enriched order to `eip.orders.enriched`.

A unit test could verify step 5 (the aggregation strategy) in isolation.
But steps 1–4 and 6–7 involve real infrastructure — Kafka serialization, Camel's `enrich()` EIP mechanics, the `direct:` route dispatch, and the Redis connection.
The integration test catches misconfiguration at any of these layers.

## Key takeaways

- **Enricher tests must seed the external data source.** The test variables for the enrichment fields must match the data that the external source returns. If the source is Redis, the catalog must be seeded with known products before the test runs.
- **Template pairs assert both preservation and addition.** The input template defines what the enricher receives; the output template defines what it must produce. The test implicitly verifies that original fields survive the merge *and* that enrichment fields are added correctly.
- **`fork=true` on the `send` handles multi-hop async processing.** The enricher involves two Camel routes (the main route and the lookup route) and a Redis call — all happening asynchronously. The non-blocking send operation gives the full pipeline time to complete and the Kafka consumer time to initialize without any racing conditions.
- **Two runtimes, one test pattern.** The test logic is identical across Quarkus and Spring Boot. The only meaningful difference is how Redis catalog seeding is triggered — automatically in Quarkus, explicitly in Spring Boot.
- **Real infrastructure catches real problems.** Connection timeouts, serialization mismatches, Redis key format issues, and Kafka consumer group configuration all surface in an integration test but are invisible in a unit test with mocked dependencies.
