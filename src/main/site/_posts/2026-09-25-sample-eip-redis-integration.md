---
layout: sample
title: Testing Redis-Backed Camel Routes with Citrus
name: redis-integration
image: /img/icons/camel.png
folder: examples/22-redis-integration
group: camel
description: Testing Redis-backed Apache Camel routes — caching, idempotent deduplication, and distributed locking — with Citrus across Quarkus and Spring Boot
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-redis-integration/
---

Redis is one of those infrastructure components that adds much value to various integration scenarios.
In an integration layer, it acts as a cache that quietly shoulders four distinct jobs: caching expensive lookups so the database is not hammered on every message, deduplicating events so downstream services are not charged twice, coordinating multiple service instances through distributed locks, and acting as a lightweight pub/sub channel for intra-service notifications.

[Apache Camel](https://camel.apache.org) integrates with Redis natively, and the patterns that Redis enables — Content Enricher with cache-aside, Idempotent Receiver with Redis-backed deduplication, and Competing Consumers with distributed locking — map cleanly to Camel's route DSL.
But integration at the boundary of a live Redis instance is exactly the kind of scenario that a unit test cannot cover.
You need a real Redis, real Kafka topics, and an integration test framework that can coordinate both sides of the exchange.

This post walks through three Redis-backed Camel routes and the [Citrus](https://citrusframework.org) integration tests that verify them end-to-end, running identically on both Quarkus and Spring Boot.
The test infrastructure follows the same pattern used across all Camel EIP examples — see the [Camel EIP examples](/samples/camel-eip/) overview page for the shared Docker Compose setup, `EipInfraSetup`, `EipTestSupport`, dependencies, and how to run the tests.
This example extends the base Compose stack with a Redis service alongside Kafka.

## The three patterns

The example covers the three most common uses of Redis in an integration layer:

| Pattern                           | Route ID              | Redis operation     | Test focus                                       |
|-----------------------------------|-----------------------|---------------------|--------------------------------------------------|
| **Content Enricher with caching** | `cached-enrichment`   | `GET` / `SETEX`     | Cache miss on first request, cache hit on second |
| **Idempotent Receiver**           | `idempotent-receiver` | `SET NX EX`         | Unique events forwarded, duplicates dropped      |
| **Distributed lock**              | `distributed-lock`    | `SET NX EX` / `DEL` | Lock acquisition and safe release                |

All three routes consume from or produce to Kafka topics, which makes the test structure consistent: send a message to a Kafka topic, wait for the Camel route to process it, and verify the output on another Kafka topic or through Camel's route statistics.

## Pattern 1 — Content Enricher with Redis caching

### The route

The first route enriches incoming order events with customer data.
High-throughput pipelines cannot afford a database call for every message, so the customer lookup is cached in Redis with a 10-minute TTL.

#### Quarkus

```java
@ApplicationScoped
public class CachingEnricherRoute extends RouteBuilder {

    @Inject
    RedisAPI redis;

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=redis-enricher")
            .routeId("cached-enrichment")
            .unmarshal().json(Map.class)
            .process(this::enrichFromCache)
            .log("Enriched order ${body[order_id]}: customer=${body[customer_name]}")
            .marshal().json()
            .to("kafka:eip.orders.enriched?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }

    private void enrichFromCache(Exchange exchange) {
        Map<String, Object> order = exchange.getIn().getBody(Map.class);
        String customerId = String.valueOf(order.get("customer_id"));
        String cacheKey = "customer:" + customerId;

        Response cached = redis.get(cacheKey).await().indefinitely();
        if (cached != null) {
            order.put("customer_name", cached.toString());
            order.put("cache_hit", true);
            return;
        }

        // Cache miss: perform lookup and populate cache
        String customerName = "Customer " + customerId;
        order.put("customer_name", customerName);
        order.put("cache_hit", false);

        // Cache for 10 minutes
        redis.setex(cacheKey, "600", customerName).await().indefinitely();
    }
}
```

#### Spring Boot

```java
@Component
public class CachingEnricherRoute extends RouteBuilder {

    @Autowired
    StringRedisTemplate redisTemplate;

    @Override
    public void configure() {
        from("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=redis-enricher")
            .routeId("cached-enrichment")
            .unmarshal().json(Map.class)
            .process(this::enrichFromCache)
            .log("Enriched order ${body[order_id]}: customer=${body[customer_name]}")
            .marshal().json()
            .to("kafka:eip.orders.enriched?brokers={% raw %}{{kafka.brokers}}{% endraw %}");
    }

    private void enrichFromCache(Exchange exchange) {
        Map<String, Object> order = exchange.getIn().getBody(Map.class);
        String customerId = String.valueOf(order.get("customer_id"));
        String cacheKey = "customer:" + customerId;

        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            order.put("customer_name", cached);
            order.put("cache_hit", true);
            return;
        }

        String customerName = "Customer " + customerId;
        order.put("customer_name", customerName);
        order.put("cache_hit", false);

        redisTemplate.opsForValue().set(cacheKey, customerName, Duration.ofSeconds(600));
    }
}
```

The route logic is identical between runtimes.
The only difference is the Redis client: Quarkus uses the Vert.x Mutiny `RedisAPI` injected via CDI, while Spring Boot uses `StringRedisTemplate` from Spring Data Redis.
In both cases, the cache-aside pattern is the same: attempt a Redis `GET`, serve the cached value if present (and mark `cache_hit: true`), otherwise perform the lookup and write the result to Redis with a `SETEX` for TTL-bounded expiration.

The enriched order — now carrying `customer_name` and a `cache_hit` flag — is serialized back to JSON and forwarded to `eip.orders.enriched`.
The `cache_hit` field is not a production concern; it exists specifically to make the caching behavior visible to tests.

### The tests

The caching enricher requires two test methods that must run in order.
The first sends an order for a customer that has never been seen — a cache miss.
The second sends a different order for the *same* customer — a cache hit.
Because Redis state is shared between test executions within the same suite, the second method can only prove a cache hit if the first method ran first and populated the cache.
`@TestClassOrder` and `@Order` annotations enforce this dependency.

#### Quarkus

```java
@QuarkusTest
@CitrusSupport
@TestClassOrder(ClassOrderer.OrderAnnotation.class)
class EipTests implements EipTestSupport {

    @CitrusResource
    TestCaseRunner t;

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    @Nested
    class CachingEnricherTest {

        @Test
        @Order(1)
        public void shouldEnrichOrderWithCacheMiss() {
            t.given(waitForCamelRouteStarted("cached-enrichment", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.placed")
                    .message()
                    .body("{\"order_id\": 1001, \"customer_id\": \"C-101\", "
                        + "\"item_sku\": \"SKU-R1\", \"quantity\": 2, \"amount\": 49.99}")
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "ORDER-1001")
            );

            t.then(
                repeatOnError()
                    .times(20)
                    .actions(
                        receive()
                            .endpoint("kafka:eip.orders.enriched"
                                + "?consumerGroup=citrus-enriched-cache-miss-group")
                            .message()
                            .body("""
                            {
                              "order_id": 1001,
                              "customer_id": "C-101",
                              "item_sku": "SKU-R1",
                              "quantity": 2,
                              "amount": 49.99,
                              "customer_name": "Customer C-101",
                              "cache_hit": false
                            }
                            """)
                    )
            );
        }

        @Test
        @Order(2)
        public void shouldEnrichOrderWithCacheHit() {
            t.given(waitForCamelRouteStarted("cached-enrichment", camelContext));

            t.when(
                send()
                    .endpoint("kafka:eip.orders.placed")
                    .message()
                    .body("{\"order_id\": 1002, \"customer_id\": \"C-101\", "
                        + "\"item_sku\": \"SKU-R2\", \"quantity\": 1, \"amount\": 29.99}")
                    .header(KafkaMessageHeaders.MESSAGE_KEY, "ORDER-1002")
            );

            t.then(
                repeatOnError()
                    .times(20)
                    .actions(
                        receive()
                            .endpoint("kafka:eip.orders.enriched"
                                + "?consumerGroup=citrus-enriched-cache-hit-group")
                            .message()
                            .body("""
                            {
                              "order_id": 1002,
                              "customer_id": "C-101",
                              "item_sku": "SKU-R2",
                              "quantity": 1,
                              "amount": 29.99,
                              "customer_name": "Customer C-101",
                              "cache_hit": true
                            }
                            """)
                    )
            );
        }
    }
}
```

**`shouldEnrichOrderWithCacheMiss`** sends order 1001 for customer `C-101`.
This customer has never been looked up, so the Redis `GET` returns nothing and the route falls through to the simulated database lookup.
The enriched output on `eip.orders.enriched` must contain `"cache_hit": false` — proving the cache was not consulted.
The `customer_name` field `"Customer C-101"` is the result of the simulated lookup and is now stored in Redis under the key `customer:C-101`.

**`shouldEnrichOrderWithCacheHit`** sends a different order — order 1002 — but for the same customer `C-101`.
This time, the Redis `GET` on `customer:C-101` returns the cached value from the first test.
The enriched output must contain `"cache_hit": true`, confirming the route served the customer name directly from Redis without performing another lookup.

Each test uses a distinct Kafka consumer group (`citrus-enriched-cache-miss-group`, `citrus-enriched-cache-hit-group`).
Without this isolation, a single consumer group tracking offsets across both tests could cause the second `receive()` call to pick up the message produced by the first test.

#### Spring Boot

```java
@SpringBootTest(classes = RedisIntegrationApplication.class,
    webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
@TestClassOrder(ClassOrderer.OrderAnnotation.class)
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class CachingEnricherTest {

        @CitrusResource
        TestCaseRunner t;

        @Test
        @Order(1)
        public void shouldEnrichOrderWithCacheMiss() {
            // Identical test logic — only the customer_id differs (C-201 instead of C-101)
            // to avoid cross-runtime state collision when both test suites are run together
        }

        @Test
        @Order(2)
        public void shouldEnrichOrderWithCacheHit() {
            // Same structure as the Quarkus test; verifies cache_hit: true for C-201
        }
    }
}
```

The test logic is identical to the Quarkus version.
Differences are limited to the class-level annotations (`@SpringBootTest`, `@CamelSpringBootTest`, `@CitrusSpringSupport`) and the `TestCaseRunner` injection point, which moves from a class-level field to nested class fields because Spring Boot's test context wires `@CitrusResource` per nested class.
The Spring Boot tests use customer `C-201` instead of `C-101` to avoid Redis key collisions if both test suites are run against the same Redis instance.

## Pattern 2 — Idempotent Receiver with Redis deduplication

### The route

The second route receives payment events from a Kafka topic and deduplicates them using Redis.
Each event carries an `event_id`.
The route uses Redis's `SET NX EX` command as a distributed `setIfAbsent` operation: if the key `idempotent:<event_id>` does not yet exist, it is created with a 24-hour TTL and the event is processed normally.
If the key already exists, the event is a duplicate and is routed to a dead-end `handle-duplicate-payment` route.

#### Quarkus

```java
@ApplicationScoped
public class IdempotentReceiverRoute extends RouteBuilder {

    @Inject
    RedisAPI redis;

    @Override
    public void configure() {
        from("kafka:eip.orders.payments?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=redis-idempotent")
            .routeId("idempotent-receiver")
            .unmarshal().json(Map.class)
            .process(this::deduplicateAndProcess)
            .choice()
                .when(header("CamelDuplicate").isEqualTo(true))
                    .to("direct:handle-duplicate-payment")
                .otherwise()
                    .log("Processing payment event ${body[event_id]} for order ${body[order_id]}")
                    .marshal().json()
                    .to("kafka:eip.orders.payment-confirmed?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
            .end();

        from("direct:handle-duplicate-payment")
            .routeId("handle-duplicate-payment")
            .log("Duplicate payment event ${body[event_id]} -- skipping");
    }

    private void deduplicateAndProcess(Exchange exchange) {
        Map<String, Object> event = exchange.getIn().getBody(Map.class);
        String eventId = String.valueOf(event.get("event_id"));
        String dedupeKey = "idempotent:" + eventId;

        // SET NX with a 24-hour TTL — returns null if the key already exists
        Response result = redis.set(List.of(dedupeKey, "1", "NX", "EX", "86400"))
            .await().indefinitely();

        boolean isDuplicate = (result == null);
        exchange.getIn().setHeader("CamelDuplicate", isDuplicate);
    }
}
```

#### Spring Boot

```java
@Component
public class IdempotentReceiverRoute extends RouteBuilder {

    @Autowired
    StringRedisTemplate redisTemplate;

    @Override
    public void configure() {
        from("kafka:eip.orders.payments?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=redis-idempotent")
            .routeId("idempotent-receiver")
            .unmarshal().json(Map.class)
            .process(this::deduplicateAndProcess)
            .choice()
                .when(header("CamelDuplicate").isEqualTo(true))
                    .to("direct:handle-duplicate-payment")
                .otherwise()
                    .log("Processing payment event ${body[event_id]} for order ${body[order_id]}")
                    .marshal().json()
                    .to("kafka:eip.orders.payment-confirmed?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
            .end();

        from("direct:handle-duplicate-payment")
            .routeId("handle-duplicate-payment")
            .log("Duplicate payment event ${body[event_id]} -- skipping");
    }

    private void deduplicateAndProcess(Exchange exchange) {
        Map<String, Object> event = exchange.getIn().getBody(Map.class);
        String eventId = String.valueOf(event.get("event_id"));
        String dedupeKey = "idempotent:" + eventId;

        // SET NX with a 24-hour TTL — returns false if the key already exists
        Boolean wasSet = redisTemplate.opsForValue()
            .setIfAbsent(dedupeKey, "1", Duration.ofSeconds(86400));

        boolean isDuplicate = !Boolean.TRUE.equals(wasSet);
        exchange.getIn().setHeader("CamelDuplicate", isDuplicate);
    }
}
```

The two implementations are semantically identical.
In Quarkus, `redis.set(List.of(dedupeKey, "1", "NX", "EX", "86400"))` returns `null` when the key already exists.
In Spring Boot, `redisTemplate.opsForValue().setIfAbsent(...)` returns `false` when the key already exists.
Both implementations set the `CamelDuplicate` header to `true` for duplicates, which the `choice()` EIP then routes to `handle-duplicate-payment`.

Routing duplicates to a named dead-end route — rather than simply filtering them with a `filter()` — is a deliberate design choice that makes the duplicate count visible through Camel's route statistics.
This is what the tests exploit.

### The tests

The idempotent receiver test suite requires two methods that run in a fixed order, using `@TestMethodOrder` and `@Order`.
The first test sends a new event and verifies it is forwarded.
The second re-sends the *same* event and verifies that the route silently drops it.

```java
@Nested
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class IdempotentReceiverTest {

    @Test
    @Order(1)
    public void shouldProcessNewPaymentEvent() {
        t.given(waitForCamelRouteStarted("idempotent-receiver", camelContext));

        t.when(
            send()
                .endpoint("kafka:eip.orders.payments")
                .message()
                .body("{\"event_id\": \"EVT-2001\", \"order_id\": 2001, \"amount\": 149.99}")
                .header(KafkaMessageHeaders.MESSAGE_KEY, "PAY-2001")
        );

        t.then(
            repeatOnError()
                .times(20)
                .actions(
                    receive()
                        .endpoint("kafka:eip.orders.payment-confirmed"
                            + "?consumerGroup=citrus-payment-confirmed-group")
                        .message()
                        .body("""
                        {
                          "event_id": "EVT-2001",
                          "order_id": 2001,
                          "amount": 149.99
                        }
                        """)
                )
        );
    }

    @Test
    @Order(2)
    public void shouldDropDuplicatePaymentEvent() {
        t.given(waitForCamelRouteStarted("idempotent-receiver", camelContext));

        t.when(
            send()
                .endpoint("kafka:eip.orders.payments")
                .message()
                .body("{\"event_id\": \"EVT-2001\", \"order_id\": 2001, \"amount\": 149.99}")
                .header(KafkaMessageHeaders.MESSAGE_KEY, "PAY-2001-DUP")
        );

        t.then(verifyCompletedExchanges("handle-duplicate-payment", 1, camelContext));
    }
}
```

**`shouldProcessNewPaymentEvent`** sends event `EVT-2001` for the first time.
The Redis `SET NX` succeeds — the key `idempotent:EVT-2001` is created and the event is forwarded to `eip.orders.payment-confirmed`.
The `receive()` block with `repeatOnError()` polls the topic up to 20 times, succeeding as soon as the confirmed message appears.

**`shouldDropDuplicatePaymentEvent`** re-sends the exact same event body with a different Kafka message key (`PAY-2001-DUP` instead of `PAY-2001` — the Kafka key has no bearing on deduplication, only `event_id` matters).
The Redis `SET NX` fails this time because `idempotent:EVT-2001` already exists from the previous test.
The route sets `CamelDuplicate: true` and routes the exchange to `direct:handle-duplicate-payment`.

The assertion for the duplicate test is `verifyCompletedExchanges("handle-duplicate-payment", 1, camelContext)`.
This polls Camel's internal route statistics and passes only when the `handle-duplicate-payment` route has completed exactly one exchange.
This approach is cleaner than using `expectTimeout()` on the output Kafka topic because it directly measures what the route did rather than waiting for an absence of output.
It also avoids adding unnecessary wait time to the test suite.
Both `waitForCamelRouteStarted` and `verifyCompletedExchanges` are shared utilities provided by the `EipTestSupport` interface — see the [Camel EIP examples](/samples/camel-eip/) overview page for their implementation.

## Understanding the Redis deduplication approach

The custom `SET NX EX` approach in these routes takes a slightly different path from Camel's built-in `idempotentConsumer()` EIP.
The built-in EIP uses a `RedisIdempotentRepository` that Camel manages internally.
The custom approach gives the route author full control over the key naming scheme, TTL, and the routing decision — including the ability to route duplicates to a named branch for counting and logging rather than simply dropping them silently.

The TTL is a deliberate design constraint.
A Redis key that never expires would grow the idempotent store indefinitely.
A 24-hour TTL (86400 seconds) bounds memory usage: events that arrive more than 24 hours after their first processing will be treated as new and processed again.
This is acceptable when Kafka's own retention window is also 24 hours — any message that could arrive as a duplicate would still be within the TTL window.

## Pattern 3 — Distributed lock for scheduled tasks

### The route

The third route addresses a problem that appears when a Camel service scales horizontally.
A nightly order export should run exactly once per night — not once per instance.
Without coordination, all three replicas of the service would fire the same timer and run the export three times.

The distributed lock pattern uses Redis's `SET NX EX` as a lease: the first instance to write the lock key wins and runs the task.
The other instances find the key already set, skip their execution, and log a message.

```java
@ApplicationScoped
public class DistributedLockRoute extends RouteBuilder {

    private static final String LOCK_KEY = "lock:nightly-order-export";
    private static final String LOCK_TTL_SECONDS = "30";

    private final String instanceId = UUID.randomUUID().toString();

    @Inject
    RedisAPI redis;

    @Override
    public void configure() {
        from("timer:nightly-export?period=60000")
            .autoStartup(enabled)
            .routeId("distributed-lock")
            .process(this::tryAcquireLock)
            .choice()
                .when(header("LockAcquired").isEqualTo(true))
                    .log("Lock acquired by instance " + instanceId)
                    .process(this::runExportTask)
                    .process(this::releaseLock)
                    .log("Nightly order export complete -- lock released")
                .otherwise()
                    .log("Lock held by another instance -- skipping nightly export")
            .end();
    }

    private void tryAcquireLock(Exchange exchange) {
        Response result = redis.set(List.of(LOCK_KEY, instanceId, "NX", "EX", LOCK_TTL_SECONDS))
            .await().indefinitely();
        exchange.getIn().setHeader("LockAcquired", result != null);
    }

    private void runExportTask(Exchange exchange) {
        int exportedCount = 42 + (int) (System.nanoTime() % 100);
        exchange.getIn().setBody(String.format(
            "{\"task\": \"nightly-order-export\", \"exported_count\": %d, \"instance\": \"%s\"}",
            exportedCount, instanceId));
    }

    private void releaseLock(Exchange exchange) {
        // Only release if we still own the lock (compare-and-delete)
        Response currentValue = redis.get(LOCK_KEY).await().indefinitely();
        if (currentValue != null && instanceId.equals(currentValue.toString())) {
            redis.del(List.of(LOCK_KEY)).await().indefinitely();
        }
    }
}
```

The lock value is the `instanceId` — a random UUID generated at startup.
After the export completes, `releaseLock` reads the current lock value and only deletes the key if it still matches `instanceId`.
This compare-and-delete prevents a stale instance from releasing a lock that has already been re-acquired by another instance after a TTL expiry.

The `autoStartup(enabled)` flag allows the distributed lock route to be disabled in tests via a configuration property (`eip.distributed.lock.enabled=false`).
This is important because the timer fires every 60 seconds — in an automated test suite, you do not want a background timer interfering with assertions that rely on route statistics.

## Why Redis integration tests are harder than they look

### Cache behavior is stateful and ordered

Testing a cache requires two messages sent to the same consumer group in a specific order.
The first message primes the cache; the second proves it.
If the test framework ran them in parallel or in an unpredictable order, the second test could either see a cache miss (if the first has not yet completed) or always see a cache hit (if the Redis state from a previous test run is still present).

Citrus's `@TestClassOrder` and `@Order` annotations enforce execution order.
The `repeatOnError()` wrapper ensures the first test does not complete until the enriched message is confirmed on the output topic — guaranteeing that the cache is warm before the second test starts.

### Verifying duplicate suppression requires route statistics

Proving that a duplicate was dropped is a negative assertion: you need to confirm that *no* output was produced.
There are two ways to verify this with Citrus.
The `expectTimeout()` approach waits for a specified duration and fails if a message arrives.
The `verifyCompletedExchanges` approach checks Camel's internal route statistics to confirm a specific route was invoked.

The route statistics approach is preferred here because the `handle-duplicate-payment` route represents a positive claim: the duplicate *was* processed — it was routed to a specific named branch.
Verifying completed exchange count is faster than waiting for a timeout and makes the test's intent explicit.

### TTL-bounded deduplication requires care with test isolation

Because Redis keys expire, the test must run within the key's TTL window — which is never a problem for 24-hour TTLs.
But shorter TTLs in distributed lock tests (30 seconds in the example) can cause intermittent failures if the test takes longer than expected.
The distributed lock route uses `autoStartup(enabled)` to disable itself during tests, avoiding uncontrolled timer firings that could compete with a lock held by the test.

For the test infrastructure setup, shared test utilities, runtime wiring, dependencies, and how to run the tests, see the [Camel EIP examples](/samples/camel-eip/) overview page.

## Key takeaways

- **Redis serves multiple roles in an integration layer — and each role needs a different test strategy.** Caching requires ordered, stateful tests that verify both a miss and a subsequent hit. Deduplication requires proving a negative — that a duplicate was absorbed and not forwarded. Distributed locking requires disabling the timer route during tests to prevent background interference.
- **`verifyCompletedExchanges` is a cleaner alternative to `expectTimeout()` for negative assertions.** Routing duplicates to a named `handle-duplicate-payment` route makes the drop count visible through Camel's route statistics. Checking that count is faster and more expressive than waiting for an absence of output on a Kafka topic.
- **The `SET NX EX` pattern is a portable distributed primitive.** Both the idempotent receiver and the distributed lock use `SET NX EX` as their core Redis operation. Quarkus and Spring Boot expose it through different client APIs — `redis.set(List.of(...))` and `redisTemplate.opsForValue().setIfAbsent(...)` — but the semantics are identical.
- **Test-ordered execution is necessary for stateful integration tests.** Cache hit tests depend on cache miss tests having completed successfully. JUnit 5's `@TestClassOrder` and `@Order` annotations, combined with Citrus's `repeatOnError()` for readiness checks, make this dependency explicit and reliable.
- **Disable timer-driven routes in tests.** Routes that fire on a schedule — like the distributed lock route — should support a configuration flag to disable them during test execution. Without this, background timer firings can increment route statistics and invalidate exchange count assertions.
- **Two runtimes, one test structure.** Whether you run on Quarkus or Spring Boot, the test shape — `given` infrastructure, `when` stimulus, `then` assertion — stays the same. The only differences are class-level annotations and the injection mechanism for `CamelContext` and `TestCaseRunner`.
