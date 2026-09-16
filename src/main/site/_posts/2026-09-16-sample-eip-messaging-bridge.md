---
layout: sample
title: Testing the Messaging Bridge Pattern with Citrus
name: messaging-bridge
image: /img/icons/camel.png
folder: examples
group: eip
description: Testing the Messaging Bridge EIP in Apache Camel with Citrus across Quarkus, Spring Boot and YAML DSL
categories: [samples]
repository: citrus-camel-eip-examples
permalink: /samples/camel-eip/messaging-bridge/
---

Enterprise systems rarely run on a single messaging platform.
One team standardizes on Apache Kafka for its event streams, while a partner organization publishes orders through Apache Pulsar.
An acquired company may send events via RabbitMQ.
Another subsystem might still use JMS queues.
In each case, messages produced on one messaging system need to flow into another — reliably, transparently, and without forcing either side to change its infrastructure.

This is the [Messaging Bridge](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingBridge.html) pattern, described in *Enterprise Integration Patterns* by Hohpe and Woolf.
A Messaging Bridge connects two messaging systems so that messages available on one are also available on the other.
It differs from a Channel Adapter, which connects a non-messaging application (like a REST API or a database) to a messaging system.
A bridge connects two systems that are *both* messaging-aware, translating between their protocols, header formats, and delivery guarantees.

[Apache Camel](https://camel.apache.org) makes implementing a Messaging Bridge deceptively simple — a single route with a messaging `from()` and a messaging `to()`.
But the simplicity of the implementation belies the complexity of testing it.
A bridge test must send a message into one messaging system, wait for it to traverse the bridge, and then verify it appears on the other messaging system — with both brokers running simultaneously, each with its own protocol and client libraries.

This is where [Citrus](https://citrusframework.org) comes in.
Citrus can produce and consume messages across different messaging technologies in the same test, manage multi-broker infrastructure with Testcontainers, and validate that a message arriving on the destination system matches what was sent on the source.
In this post, we walk through a complete example that tests a bidirectional Messaging Bridge between Pulsar and Kafka, using Citrus on Quarkus, Spring Boot, and the Camel YAML DSL.

# The Camel route under test

The scenario models a partnership integration.
A partner organization runs on Apache Pulsar and publishes orders to a `partner.orders.placed` topic.
Your domain runs on Kafka and expects orders on the `eip.orders.placed` topic.
The Messaging Bridge consumes orders from the Pulsar topic and forwards them to the Kafka topic.

A second bridge route handles the reverse direction: shipping events published to a Kafka topic `eip.shipping.scheduled` are forwarded to a Pulsar topic of the same name, so the partner system can track shipment progress.

Here is the Camel route on Quarkus:

```java
@ApplicationScoped
public class MessagingBridgeRoute extends RouteBuilder {

    @Override
    public void configure() {
        // Pulsar → Kafka: partner orders arrive on Pulsar, bridged into Kafka
        from("pulsar:persistent://public/default/partner.orders.placed"
                + "?subscriptionName=kafka-bridge"
                + "&subscriptionType=Exclusive")
            .routeId("messaging-bridge-pulsar-to-kafka")
            .log("Messaging Bridge — Pulsar → Kafka: ${body}")
            .to("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
            .log("Messaging Bridge — order forwarded to Kafka");

        // Kafka → Pulsar: shipping events bridged to Pulsar for partner systems
        from("kafka:eip.shipping.scheduled?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=pulsar-bridge")
            .routeId("messaging-bridge-kafka-to-pulsar")
            .log("Messaging Bridge — Kafka → Pulsar: ${body}")
            .to("pulsar:persistent://public/default/eip.shipping.scheduled"
                + "?producerName=kafka-bridge")
            .log("Messaging Bridge — shipping event forwarded to Pulsar");
    }
}
```

The route looks remarkably simple — each bridge direction is just a `from()` and a `to()` — but there is important behavior beneath the surface.

First, the Pulsar consumer uses `subscriptionType=Exclusive`, which means only one consumer instance can subscribe at a time.
This prevents duplicate bridging when multiple application instances are running — only one instance receives each message.

Second, the `subscriptionName=kafka-bridge` creates a named, durable subscription on Pulsar.
If the bridge restarts, it resumes from where it left off rather than replaying from the beginning of the topic or losing unprocessed messages.

Third, and most critically for reliability: Camel's synchronous processing model is what makes the bridge safe.
The `to("kafka:...")` call blocks until the Kafka producer receives an acknowledgement from the broker.
Only after Kafka confirms receipt does the Pulsar consumer acknowledge its message.
If the Kafka publish fails, the Pulsar message is not acknowledged, so it will be redelivered on the next attempt.
This is the essential guarantee of a Messaging Bridge — no message is lost in transit between the two systems.

On Spring Boot, the route is identical except for the annotation:

```java
@Component
public class MessagingBridgeRoute extends RouteBuilder {

    @Override
    public void configure() {
        // Pulsar → Kafka: partner orders arrive on Pulsar, bridged into Kafka
        from("pulsar:persistent://public/default/partner.orders.placed"
                + "?subscriptionName=kafka-bridge"
                + "&subscriptionType=Exclusive")
            .routeId("messaging-bridge-pulsar-to-kafka")
            .log("Messaging Bridge — Pulsar → Kafka: ${body}")
            .to("kafka:eip.orders.placed?brokers={% raw %}{{kafka.brokers}}{% endraw %}")
            .log("Messaging Bridge — order forwarded to Kafka");

        // Kafka → Pulsar: shipping events bridged to Pulsar for partner systems
        from("kafka:eip.shipping.scheduled?brokers={% raw %}{{kafka.brokers}}{% endraw %}&groupId=pulsar-bridge")
            .routeId("messaging-bridge-kafka-to-pulsar")
            .log("Messaging Bridge — Kafka → Pulsar: ${body}")
            .to("pulsar:persistent://public/default/eip.shipping.scheduled"
                + "?producerName=kafka-bridge")
            .log("Messaging Bridge — shipping event forwarded to Pulsar");
    }
}
```

The same routing logic expressed in the Camel YAML DSL:

```yaml
# Pulsar → Kafka: partner orders arrive on Pulsar, bridged into Kafka
- route:
    id: messaging-bridge-pulsar-to-kafka
    from:
      uri: "pulsar:persistent://public/default/partner.orders.placed"
      parameters:
        subscriptionName: kafka-bridge
        subscriptionType: Exclusive
      steps:
        - log: "Messaging Bridge — Pulsar → Kafka: ${body}"
        - to:
            uri: "kafka:eip.orders.placed"
            parameters:
              brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
        - log: "Messaging Bridge — order forwarded to Kafka"

# Kafka → Pulsar: shipping events bridged to Pulsar for partner systems
- route:
    id: messaging-bridge-kafka-to-pulsar
    from:
      uri: "kafka:eip.shipping.scheduled"
      parameters:
        brokers: "{% raw %}{{kafka.brokers}}{% endraw %}"
        groupId: pulsar-bridge
      steps:
        - log: "Messaging Bridge — Kafka → Pulsar: ${body}"
        - to:
            uri: "pulsar:persistent://public/default/eip.shipping.scheduled"
            parameters:
              producerName: kafka-bridge
        - log: "Messaging Bridge — shipping event forwarded to Pulsar"
```

All three variants express the same bidirectional bridge: orders flow from Pulsar to Kafka, shipping events flow from Kafka to Pulsar.

# What makes testing a messaging bridge different

Testing a Messaging Bridge is fundamentally different from testing a route that stays within a single broker.
With a Content-Based Router or a Splitter, you send to a Kafka topic and receive from a Kafka topic — the test operates within one messaging system.
With a bridge, the test must cross the protocol boundary: send on one technology, receive on another.

This creates three distinct challenges.

**Multi-broker infrastructure**: the test environment must run both a Kafka broker and a Pulsar broker simultaneously, each with its own startup sequence and health checks.
The test cannot begin until both brokers are healthy and the bridge route has connected to both.

**Cross-protocol send and receive**: the test must produce a message using Pulsar's client protocol and consume the bridged result using Kafka's consumer protocol (or vice versa).
The Citrus framework must support both protocols in the same test case, with endpoint URIs that speak each broker's native language.

**Content fidelity across systems**: the test must verify that the message content survives the bridge intact.
Pulsar and Kafka have different header/property models, different serialization defaults, and different metadata.
The bridge must preserve the application payload even though the envelope changes.

# The test infrastructure

The bridge test requires both a Kafka broker and a Pulsar broker.
The Docker Compose file provisions both alongside a Kafka UI for visual debugging:

```yaml
services:
  kafka:
    image: docker.io/apache/kafka:4.3.1
    ports:
      - "9092:9092"
    environment:
      - KAFKA_NODE_ID=1
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - KAFKA_AUTO_CREATE_TOPICS_ENABLE=true
    healthcheck:
      test: ["CMD-SHELL", "nc -z localhost 9092"]
      interval: 5s
      timeout: 5s
      retries: 12
      start_period: 30s

  pulsar:
    image: docker.io/apachepulsar/pulsar:4.2.4
    command: bin/pulsar standalone --advertised-address localhost
    ports:
      - "6650:6650"
      - "8080:8080"
    environment:
      - PULSAR_MEM=-Xms512m -Xmx1024m -XX:MaxDirectMemorySize=512m
      - PULSAR_STANDALONE_USE_ZOOKEEPER=0
    healthcheck:
      test: ["CMD-SHELL", "bin/pulsar-admin brokers healthcheck"]
      interval: 10s
      timeout: 5s
      retries: 15
      start_period: 60s
```

Note that Pulsar's startup is significantly slower than Kafka's — its health check uses a 60-second `start_period` with up to 15 retries.
This affects how the test infrastructure setup waits for readiness, which we will see in the Citrus configuration below.

Citrus manages the lifecycle of these containers with Testcontainers.
On Quarkus, the setup uses `@CitrusConfiguration` with `@BindToRegistry`:

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

The `waitFor().http()` action probes the Kafka UI at port 8090, which only becomes available after Kafka is healthy.
This provides a readiness gate for the Kafka side.
The Pulsar readiness is handled implicitly — the Camel route's Pulsar consumer will retry its connection until the broker is available, and the `waitForCamelRouteStarted` utility (used in each test) ensures the route is fully connected before the test proceeds.

# The message template

All bridge tests use a shared JSON template stored in `src/test/resources/templates/order.json`:

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

Citrus resolves the `${variable}` placeholders at runtime from test variables.
This template is used for both sending and receiving — the same template with the same variables validates that the message content survives the bridge unchanged.

# Testing the Pulsar-to-Kafka bridge

The core test sends an order to the Pulsar topic and verifies it arrives on the Kafka topic.
This exercises the `messaging-bridge-pulsar-to-kafka` route.
Here is the test on Quarkus:

```java
@Nested
class MessagingBridgePulsarToKafkaTest {

    @Test
    public void shouldBridgeOrderFromPulsarToKafka() {
        t.given(
            createVariables()
                .variable("id", "citrus:randomNumber(4)")
                .variable("amount", 200)
                .variable("status", "partner-placed")
                .variable("priority", "STANDARD")
        );

        t.given(waitForCamelRouteStarted(
            "messaging-bridge-pulsar-to-kafka", camelContext));

        t.when(
            camel()
                .send()
                .endpoint(CamelSupport.camel().endpoints()
                        .pulsar("persistent://public/default/partner.orders.placed")
                        .serviceUrl("pulsar://localhost:6650")
                        .producerName("citrus-test")::getRawUri)
                .fork(true)
                .message()
                .body(Resources.create("templates/order.json"))
        );

        t.then(
            repeatOnError()
                .until((i, context) -> i > 10)
                .autoSleep(Duration.ofSeconds(1))
                .actions(
                    receive()
                        .endpoint("kafka:eip.orders.placed?consumerGroup=citrus-placed-group")
                        .message()
                        .body(Resources.create("templates/order.json"))
                )
        );
    }
}
```

There is a lot happening in this test, so let's break it down.

The `given` phase creates test variables — a random order ID, an amount, a status of `partner-placed` to indicate this order originates from the partner system, and a standard shipping priority.
Then it waits for the bridge route to be fully started and connected to both brokers.

The `when` phase sends the order to Pulsar.
Citrus itself does not provide an endpoint implementation for Pulsar.
This is where the combination of Citrus and Apache Camel is essential.
Citrus is able to use the Camel implementation for Pulsar to send and receive messages as part of the test.
This works for all endpoint components that Apache Camel provides, that's fantastic.  

This is the cross-protocol step — notice how the endpoint is constructed using Citrus's Camel endpoint builder: `CamelSupport.camel().endpoints().pulsar(...)`.
This builder constructs the full Pulsar endpoint URI with the service URL (`pulsar://localhost:6650`) and a producer name for the test.
The `::getRawUri` method reference extracts the raw URI string that the Citrus send action needs.

The `fork(true)` setting is important here.
It tells Citrus to send the message asynchronously — fire the Pulsar publish and immediately proceed to the receive phase without waiting for the send to complete.
Without forking, the test would send, wait for the Pulsar acknowledgement, and then start listening on Kafka — but by that time, the bridge may have already forwarded the message.
Forking the send operation overlaps the publish and consume phases, eliminating this race condition.

The `then` phase consumes from the Kafka output topic using `repeatOnError` for resilient polling.
The test retries up to 10 times with one-second pauses, waiting for the bridged message to appear on `eip.orders.placed`.
The receive action uses a dedicated consumer group (`citrus-placed-group`) to avoid interfering with the bridge route's own consumer groups.

The validation uses the same `order.json` template with the same variables that were used for sending.
This proves that the message content survived the Pulsar-to-Kafka bridge intact — the order ID, amount, status, and all other fields match exactly what was sent to Pulsar.

# Waiting for the Camel route

Bridge routes connect to two brokers, so the startup wait is especially important.
The `waitForCamelRouteStarted` utility uses Camel's Control Bus to poll the route status:

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
Once the route reports `Started`, it waits an additional five seconds for both the Pulsar subscription and the Kafka consumer to fully connect.
The 20-retry count and 5-second post-start sleep are more generous than what a single-broker test would need — the bridge route can't report `Started` until both its source consumer and destination producer are initialized, and the Pulsar consumer in particular can take several seconds to establish its exclusive subscription.

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
    class MessagingBridgePulsarToKafkaTest {
        // ... test methods
    }
}
```

On Spring Boot, the test uses `@CitrusSpringSupport` alongside Spring Boot and Camel test annotations:

```java
@SpringBootTest(classes = ChannelInfraApplication.class,
    webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
@CamelSpringBootTest
@CitrusSpringSupport
@ContextConfiguration(classes = { EipInfraSetup.class, CitrusSpringConfig.class })
class EipTests implements EipTestSupport {

    @Autowired
    CamelContext camelContext;

    @Nested
    class MessagingBridgePulsarToKafkaTest {

        @CitrusResource
        TestCaseRunner t;

        // ... test methods
    }
}
```

On Spring Boot, the `TestCaseRunner` is declared in the `@Nested` inner class, while on Quarkus it lives at the outer class level — a structural difference driven by how each test framework manages injection scopes.
The actual test methods are identical on both runtimes.

# Testing with the YAML DSL

The Camel CLI supports running routes from YAML files and testing them with Citrus YAML test definitions.
The Messaging Bridge test in YAML DSL covers both bridge directions in sequence:

```yaml
name: messaging-bridge-test
description: >-
  Test verifying the Messaging Bridge pattern — bidirectional
  Pulsar ↔ Kafka bridging
variables:
  - name: kafka.broker
    value: localhost:9092
  - name: order.id
    value: "citrus:randomNumber(4)"
  - name: order.status
    value: "partner-placed"
actions:
  - testcontainers:
      compose:
        up:
          file: "_infra/compose.yaml"
  - waitFor:
      timeout: "25000"
      http:
        url: http://localhost:8090
  - camel:
      jbang:
        run:
          integration:
            name: "messaging-bridge"
            file: "../messaging-bridge.yaml"
            systemProperties:
              file: "../application.properties"

  # --- Pulsar → Kafka ---
  - send:
      endpoint: >-
        camel:pulsar:persistent://public/default/partner.orders.placed?serviceUrl=pulsar://localhost:6650&producerName=citrus-bridge-test
      fork: true
      message:
        body:
          resource:
            file: "templates/order.json"
  - camel:
      jbang:
        verify:
          integration: "messaging-bridge"
          logMessage: "Messaging Bridge — Pulsar → Kafka"
  - camel:
      jbang:
        verify:
          integration: "messaging-bridge"
          logMessage: "order forwarded to Kafka"

  # --- Kafka → Pulsar ---
  - createVariables:
      variables:
        - name: order.id
          value: "citrus:randomNumber(4)"
        - name: order.status
          value: "shipped"
  - send:
      endpoint: >-
        kafka:eip.shipping.scheduled?server=${kafka.broker}
      message:
        body:
          resource:
            file: "templates/order.json"
  - camel:
      jbang:
        verify:
          integration: "messaging-bridge"
          logMessage: "Messaging Bridge — Kafka → Pulsar"
  - camel:
      jbang:
        verify:
          integration: "messaging-bridge"
          logMessage: "shipping event forwarded to Pulsar"
```

The YAML test is self-contained: it starts both brokers via Testcontainers, waits for Kafka UI readiness, launches the Camel integration with `camel:jbang:run`, and tests both bridge directions.

The Pulsar-to-Kafka test sends an order to the Pulsar topic with `fork: true` and then uses `camel:jbang:verify` to check the running integration's log for the expected bridge messages.
The verify action waits until the log message appears, serving as both a synchronization point and a secondary confirmation that the bridge processed the message.

The Kafka-to-Pulsar test creates fresh variables (a new order ID with status `shipped`), sends to the Kafka topic, and verifies the bridge log output for the reverse direction.
Notice that the YAML test uses `camel:jbang:verify` on the log output rather than consuming from the Pulsar output topic — this avoids the complexity of setting up a Pulsar consumer subscription in the test itself while still confirming the bridge processed the message end-to-end.

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

The `citrus-camel` module is especially important for the bridge test.
It provides both the Camel Control Bus integration used in `waitForCamelRouteStarted` and the `CamelSupport.camel().endpoints().pulsar(...)` builder used to construct the Pulsar send endpoint.
This means the test does not need a separate Pulsar client library — it leverages Camel's own Pulsar component through the Citrus-Camel integration to produce test messages.

# Running the tests

Make sure Docker (or Podman) is running, then execute:

```bash
# Quarkus
cd examples/06-channel-infra/quarkus
mvn verify

# Spring Boot
cd examples/06-channel-infra/spring-boot
mvn verify
```

The test suite starts Kafka, Pulsar, and PostgreSQL via Testcontainers, boots the Camel application, runs the bridge test, and tears everything down.
The Pulsar broker takes longer to start than Kafka — expect 60 to 90 seconds for the infrastructure to become fully ready before the first test sends a message.

For the YAML DSL variant, install the [Camel CLI](https://camel.apache.org/manual/camel-jbang.html) and run:

```bash
cd examples/06-channel-infra/yaml-dsl
camel test test/messaging-bridge.citrus.it.yaml
```

# Key takeaways

Testing a Messaging Bridge means proving that messages cross the boundary between two messaging systems with their content intact.
Unlike single-broker tests, a bridge test must manage multi-broker infrastructure, send and receive across different protocols, and handle the longer startup times that come with running multiple messaging platforms.

Here is what Citrus brings to this problem:

- **Citrus & Apache Camel components** — Whenever there is a transport or messaging protocol that is not supported by Citrus you can use the Apache Camel component implementation. 
- **Cross-protocol send and receive** — The test sends to Pulsar using the `CamelSupport.camel().endpoints().pulsar(...)` endpoint builder and receives from Kafka using the standard Kafka endpoint. Citrus handles both protocols in the same test case without requiring separate client libraries.
- **`fork(true)` for overlapping operations** — Forking the send ensures the test starts listening on the destination system before the bridge finishes processing, eliminating race conditions where the bridged message arrives before the test consumer is ready.
- **`repeatOnError` for async polling** — The bridge introduces latency from two broker hops (source acknowledge, destination publish). `repeatOnError` polls the destination topic and succeeds as soon as the bridged message arrives, keeping tests fast without brittle fixed sleeps.
- **Template-based content fidelity** — The same `order.json` template is used for both sending and receiving, proving the message payload survives the protocol translation unchanged.
- **`camel:jbang:verify` for log-based confirmation** — In the YAML DSL test, log verification provides a secondary confirmation that the bridge processed the message, independent of consuming from the destination topic.
- **Multi-runtime portability** — The same test logic runs on Quarkus, Spring Boot, and the Camel CLI with YAML DSL. Only the wiring annotations and infrastructure registration change; the bridge verification logic stays identical.

The complete source code for this example is available on GitHub:

- [Quarkus variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/06-channel-infra/quarkus)
- [Spring Boot variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/06-channel-infra/spring-boot)
- [YAML DSL variant](https://github.com/citrusframework/citrus-camel-eip-examples/tree/main/examples/06-channel-infra/yaml-dsl)

Give it a try, and let us know what you think!
