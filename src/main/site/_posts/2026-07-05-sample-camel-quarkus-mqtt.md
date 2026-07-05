---
layout: sample
title: Apache Camel Quarkus - MQTT to Kafka Routing
name: camel-mqtt
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel MQTT-to-Kafka routing with Content Based Router in Quarkus using Citrus
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-mqtt/
---

Real-world integration often spans multiple messaging protocols. A sensor publishes temperature readings over MQTT, your application picks them up, decides whether the reading is warm or cold, and routes it to the appropriate Kafka topic for downstream processing. 

Apache Camel makes building this kind of multi-protocol bridge straightforward with its extensive component library and enterprise integration patterns.

Testing it is the essential yet hard part. You need an MQTT broker, a Kafka broker, topics created, messages sent over one protocol and verified on another — all coordinated in a single test. And what if your test framework does not have native MQTT support?

This post walks you through testing an Apache Camel route that bridges MQTT and Kafka in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus leverages Camel's own MQTT component to send test messages, provisions both brokers with simple annotations, and validates that messages land on the correct Kafka topic based on content-based routing. 

By the end you will have a recipe for testing multi-protocol Camel routes — even when Citrus does not natively support one of the protocols.

# The application under test

The example application implements a temperature routing service. It consumes JSON temperature readings from an MQTT topic, extracts the numeric value, and routes the message to one of two Kafka topics depending on whether the temperature is warm or cold.

```
MQTT Topic (temperature) --> JQ Transform (.value) --> Content Based Router --> Kafka (temperature-warm / temperature-cold)
```

The entire logic lives in a single Camel route:

```java
public class Routes extends EndpointRouteBuilder {

    @Override
    public void configure() throws Exception {
        from("kamelet:mqtt5-source?brokerUrl={{mqtt.broker.url}}&topic={{mqtt.topic}}")
            .transform().jq(".value")
            .convertBodyTo(Integer.class)
            .choice()
                .when().simple("${body} > 20")
                    .log("Warm temperature: ${body}")
                    .to(kafka("temperature-warm"))
                .otherwise()
                    .log("Cold temperature: ${body}")
                    .to(kafka("temperature-cold"))
            .end();
    }
}
```

There is a lot happening in these few lines. The route uses a **Kamelet** — a pre-built, reusable Camel connector — to consume MQTT messages from the `temperature` topic. It then applies a **JQ expression** (`.value`) to extract the numeric value from a JSON body like `{"value": 25}`. After converting the extracted value to an integer, the **Content Based Router** EIP evaluates whether the temperature exceeds 20 and routes accordingly: warm readings go to the `temperature-warm` Kafka topic, cold readings go to `temperature-cold`.

The route extends `EndpointRouteBuilder` for type-safe access to the `kafka()` endpoint DSL. The MQTT broker URL and topic are externalized as Camel property placeholders, resolved from Quarkus `application.properties`:

```properties
mqtt.broker.url=tcp://localhost:1883
mqtt.topic=temperature
kafka.bootstrap.servers=localhost:9092
```

That is the entire application — one route that bridges two protocols with content-based routing. Now let's verify all of it.

# Adding Citrus to the project

This example needs more Citrus modules than the simpler samples because it spans two messaging protocols. Add these test-scoped dependencies to the Maven POM:

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
    <artifactId>citrus-junit-jupiter</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Here is what each module brings to the table:

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle.
- **citrus-camel** provides the `CamelSupport.camel()` DSL that lets you use any Camel component as a Citrus endpoint — this is how we send MQTT messages.
- **citrus-kafka** provides Kafka endpoint implementations for receiving and validating messages.
- **citrus-testcontainers** adds container provisioning annotations for Kafka and generic containers.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

The key module here is **citrus-camel**. Citrus does not have a native MQTT endpoint, but it does not need one — it can delegate to Camel's `paho-mqtt5` component instead. This pattern extends Citrus's reach to any protocol that Camel supports.

# Provisioning the test infrastructure

This test needs two brokers: an MQTT broker for the input side and a Kafka broker for the output side. Citrus provisions both with annotations on the test class.

## The Kafka broker

The Kafka side uses the familiar `@KafkaContainerSupport` annotation with a lifecycle listener that creates the topics:

```java
@KafkaContainerSupport(port = 9092, version = "4.2.0",
        containerLifecycleListener = QuarkusApplicationTest.KafkaConfigurer.class)
class QuarkusApplicationTest {

    public static class KafkaConfigurer
            implements ContainerLifecycleListener<KafkaContainer> {
        @Override
        public Map<String, String> started(KafkaContainer container) {
            try (Admin adminClient = Admin.create(
                    Collections.singletonMap(
                        AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG,
                        container.getBootstrapServers()))) {

                adminClient.createTopics(Set.of(
                    new NewTopic("temperature-warm", 1, (short) 1),
                    new NewTopic("temperature-cold", 1, (short) 1)
                )).all().get();
            }

            return Collections.emptyMap();
        }
    }
}
```

This is the same pattern you see in the [Camel Kafka sample](/samples/camel-quarkus-kafka/) — the listener creates both output topics before the application starts.

## The MQTT broker

Citrus does not ship a dedicated MQTT container annotation, but it provides `@TestcontainersSupport` with a `GenericContainerProvider` that works with any Docker image. For MQTT, we use the Eclipse Mosquitto broker:

```java
@TestcontainersSupport(
        containerProvider = QuarkusApplicationTest.MosquittoContainerProvider.class,
        containerLifecycleListener = QuarkusApplicationTest.MosquittoConfigurer.class)
class QuarkusApplicationTest {

    public static class MosquittoContainerProvider implements GenericContainerProvider {
        @Override
        public GenericContainer<?> create() {
            return new GenericContainer<>("eclipse-mosquitto:latest")
                    .withExposedPorts(MOSQUITTO_PORT)
                    .withCommand("mosquitto", "-c",
                            "/mosquitto-no-auth.conf", "-p",
                            String.valueOf(MOSQUITTO_PORT))
                    .waitingFor(Wait.forListeningPort());
        }
    }

    public static class MosquittoConfigurer
            implements ContainerLifecycleListener<GenericContainer<?>> {
        @Override
        public Map<String, String> started(GenericContainer<?> container) {
            String brokerUrl = "tcp://%s:%d".formatted(
                    container.getHost(),
                    container.getMappedPort(MOSQUITTO_PORT));
            return Map.of("mqtt.broker.url", brokerUrl);
        }
    }
}
```

The `GenericContainerProvider` creates a Mosquitto container with anonymous access enabled. The `MosquittoConfigurer` is where the integration magic happens — its `started()` method returns a map with the key `mqtt.broker.url` and the dynamic broker URL as the value. Citrus injects this property into the Quarkus application context, which means the Camel route's `{{mqtt.broker.url}}` placeholder resolves to the test container automatically.

This is a powerful pattern. Any Docker image can become test infrastructure with a container provider and a lifecycle listener. The listener's return map lets you inject connection properties into the application without touching configuration files.

# Configuring Kafka endpoints

The output side of the test uses standard Citrus Kafka endpoints to receive and validate messages:

```java
@CitrusEndpoint
@KafkaEndpointConfig(topic = "temperature-warm", consumerGroup = "citrus-consumer-1")
KafkaEndpoint temperatureWarm;

@CitrusEndpoint
@KafkaEndpointConfig(topic = "temperature-cold", consumerGroup = "citrus-consumer-2")
KafkaEndpoint temperatureCold;
```

Each endpoint listens on one of the two output topics. The `consumerGroup` parameter assigns a unique consumer group to each endpoint, which ensures that both consumers can read from their respective topics independently. The bootstrap server address is automatically resolved from the `@KafkaContainerSupport` configuration.

# Sending MQTT messages through Camel

This is where the example gets interesting. Citrus has no native MQTT endpoint, so instead of building one, we leverage Camel's `paho-mqtt5` component directly from the test:

```java
runner.when(
    camel()
        .send()
        .endpoint("paho-mqtt5:temperature?brokerUrl={{mqtt.broker.url}}")
        .message()
        .fork(true)
        .body("{\"value\": 25}")
);
```

The `camel()` DSL gives Citrus access to any Camel component. Here it uses `paho-mqtt5` to publish a JSON message to the `temperature` MQTT topic. The `brokerUrl` placeholder resolves through the same Camel property mechanism as the application route — both point to the Mosquitto test container.

The `fork(true)` option is important. MQTT publishing can block while waiting for broker acknowledgment, and the test thread needs to proceed to the verification step. Forking sends the message asynchronously so the test can continue without waiting for the MQTT handshake to complete.

This approach has a broader implication: any of Camel's 300+ components can be used as a Citrus test endpoint. If your application consumes from a protocol that Citrus does not natively support — MQTT, AMQP, gRPC, FTP — you can send test messages through Camel's implementation of that protocol. The Citrus-Camel integration bridges the gap.

# Writing the tests

With both brokers running and the endpoints configured, the tests verify that the Content Based Router sends messages to the correct Kafka topic.

## Testing the warm path

```java
@Test
void shouldRouteWarmTemperature() {
    runner.when(
        camel()
            .send()
            .endpoint("paho-mqtt5:temperature?brokerUrl={{mqtt.broker.url}}")
            .message()
            .fork(true)
            .body("{\"value\": 25}")
    );

    runner.then(
        receive()
            .endpoint(temperatureWarm)
            .message()
            .body("25")
    );
}
```

The test sends a JSON message with a temperature value of 25 to the MQTT `temperature` topic. The Camel route consumes it, extracts the value with JQ, evaluates `25 > 20` as true, and routes it to `temperature-warm`. The `receive()` action verifies that the message `25` (the extracted integer, not the original JSON) arrives on the warm topic.

## Testing the cold path

```java
@Test
void shouldRouteColdTemperature() {
    runner.when(
        camel()
            .send()
            .endpoint("paho-mqtt5:temperature?brokerUrl={{mqtt.broker.url}}")
            .message()
            .fork(true)
            .body("{\"value\": 15}")
    );

    runner.then(
        receive()
            .endpoint(temperatureCold)
            .message()
            .body("15")
    );
}
```

Same structure, different value. A temperature of 15 evaluates `15 > 20` as false, so the Content Based Router sends it to `temperature-cold`. Together, these two tests cover both branches of the routing logic.

Notice that the receive assertions validate `"25"` and `"15"` — not the original JSON `{"value": 25}`. This is because the Camel route applies a JQ transformation that strips the JSON wrapper and extracts just the numeric value before routing to Kafka. The test validates what actually arrives on the Kafka topic, not what was sent to MQTT.

# How it all fits together

When you run the tests with `./mvnw clean test`, here is the sequence of events:

1. **`@TestcontainersSupport` launches** a Mosquitto MQTT broker container and injects the broker URL into Quarkus properties.
2. **`@KafkaContainerSupport` launches** a Kafka broker container on port 9092.
3. **The Kafka lifecycle listener** creates the `temperature-warm` and `temperature-cold` topics.
4. **Quarkus starts** the application in test mode with both broker URLs configured.
5. **Apache Camel discovers** the route and starts the `mqtt5-source` Kamelet consumer on the `temperature` MQTT topic.
6. **The test sends** a JSON message via Camel's `paho-mqtt5` component to the MQTT broker.
7. **The Camel route consumes** the MQTT message, extracts the value with JQ, and routes it to the appropriate Kafka topic.
8. **Citrus receives** the message from the Kafka topic and validates the body.
9. **Both containers are stopped** and cleaned up automatically.

The critical insight is that three separate concerns — MQTT consumption, content-based routing, and Kafka production — are all verified in a single test. The test sends input over one protocol and verifies output on another, covering the entire pipeline including the transformation and routing logic.

There is one piece of test configuration worth mentioning. Because Citrus and Quarkus share the same classpath, you need to allow split packages from the Citrus framework:

```properties
quarkus.arc.ignored-split-packages=org.citrusframework.*
```

This avoids CDI bean discovery conflicts and is a one-line addition that you set once and forget.

# When Citrus does not support a protocol

This example demonstrates an important pattern: you do not need Citrus to have a native endpoint for every protocol your application uses. The Citrus-Camel integration lets you tap into Camel's component library for any protocol that Citrus does not cover natively.

The pattern is straightforward:

1. Add the `citrus-camel` dependency and the relevant Camel component (e.g., `camel-paho-mqtt5`).
2. Use `camel().send().endpoint("component:destination?options")` to send messages through that component.
3. Use `camel().receive().endpoint(...)` if you also need to consume from that protocol.

This works because Citrus delegates the protocol handling entirely to Camel. You get Camel's mature, production-tested client implementation with Citrus's test DSL and validation capabilities on top.

# Where to go from here

This example covers the fundamentals of multi-protocol testing, but there are several directions you can take it:

- **More routing branches**: Add additional temperature ranges (hot, freezing) and verify that each message lands on the correct topic. The Content Based Router supports any number of `when` clauses.
- **JSON validation on Kafka**: Add the `citrus-validation-json` module to validate structured JSON payloads on the Kafka output topics instead of plain strings.
- **Error scenarios**: Test what happens when the MQTT message contains invalid JSON or a non-numeric value. Camel's error handling and dead-letter channels can be verified the same way.
- **Additional protocols**: Apply the Citrus-Camel pattern to other protocols — AMQP, FTP, gRPC, or any of Camel's 300+ components. The approach stays the same regardless of the protocol.
- **Simpler transports**: If your Camel route uses JMS or Kafka without MQTT, the [Camel JMS sample](/samples/camel-quarkus-jms/) and [Camel Kafka sample](/samples/camel-quarkus-kafka/) on this site demonstrate setups with fewer moving parts.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers Kafka endpoints, Testcontainers integration, Apache Camel support, and the many other messaging transports that Citrus supports.
