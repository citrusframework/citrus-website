---
layout: samples
title: Camel EIP
permalink: /samples/camel-eip/
name: sample-camel-eip
---

<article>
  <h2>
    Camel EIP Examples
  </h2>
  <div class="right">
    <a href="https://github.com/citrusframework/citrus-camel-eip-examples" target="_blank">
      <i class="fa fa-pencil"></i>&nbsp;View source code
    </a>
  </div>
  <span class="post-category">
    <span class="label">
      Setup
    </span>
  </span>
  <div class="post-content" markdown="1">

All Camel EIP samples share the same test infrastructure, utility classes, and build setup.
This page documents the common pieces so the individual sample posts can focus on the EIP-specific test logic.

The source code for all EIP samples is on [GitHub](https://github.com/citrusframework/citrus-camel-eip-examples){:target="_blank"}.

### Test infrastructure

Every EIP sample runs a Kafka broker as the messaging backbone.
The tests use Citrus Testcontainers Docker Compose lifecycle management to start the infrastructure before the test suite and tear it down afterwards.

The Docker Compose file provisions a single-node Kafka broker in KRaft mode (no ZooKeeper) alongside a Kafka UI for visual debugging:

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

  kafka-ui:
    image: docker.io/provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    depends_on:
      kafka:
        condition: service_healthy
```

Some EIP samples add additional services to the same Compose file depending on what the pattern requires:

- **PostgreSQL** — used by the Aggregator (persistent aggregation repository), Polling Consumer (SQL polling), Idempotent Receiver (JDBC deduplication repository), and Outbox Pattern (transactional writes). Each mounts an init script to create the required schema at startup:

```yaml
  postgres:
    image: docker.io/library/postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=eipdb
      - POSTGRES_USER=eipuser
      - POSTGRES_PASSWORD=eippass
    volumes:
      - ./postgres/init-schemas.sql:/docker-entrypoint-initdb.d/01-init-schemas.sql:Z
```

- **Apache Pulsar** — used by the Messaging Bridge (bidirectional Pulsar-Kafka bridging). Pulsar's startup is significantly slower than Kafka's — its health check uses a 60-second `start_period` with up to 15 retries:

```yaml
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

The `EipInfraSetup` class ties the Docker Compose lifecycle to the test suite.
A `BeforeSuite` action starts the compose stack and waits for readiness; an `AfterSuite` tears it down.

On **Quarkus** this uses `@CitrusConfiguration` with `@BindToRegistry`:

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

On **Spring Boot** the same logic uses `@Configuration` and `@Bean`:

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
This provides a readiness gate for the test suite.

### Shared test utility

All EIP test classes implement the `EipTestSupport` interface that provides three shared helper methods:

```java
public interface EipTestSupport {

    default void waitForCamelRouteStarted(TestContext context, String routeId) {
        // Wait for Camel route to be started
        await()
            .atMost(Duration.ofSeconds(30))
            .pollInterval(Duration.ofMillis(500))
            .untilAsserted(() ->
                context.run(camel()
                    .jmx()
                    .route(routeId)
                    .verify()
                    .status("Started")
                )
            );
    }

    default void verifyCompletedExchanges(
            TestContext context, String routeId, int expected) {
        // Verify number of completed exchanges on the route
        context.run(camel()
            .jmx()
            .route(routeId)
            .verify()
            .exchangeCompleted(expected)
        );
    }

    default void verifyRouteStats(
            TestContext context, String routeId,
            int completed, int failed) {
        // Verify route completion and failure counts
        context.run(camel()
            .jmx()
            .route(routeId)
            .verify()
            .exchangeCompleted(completed)
            .exchangeFailed(failed)
        );
    }
}
```

The `waitForCamelRouteStarted` method uses Awaitility to poll until the named Camel route is in `Started` state — essential because there is a startup delay between the Kafka broker becoming available and the Camel route fully initializing.

The `verifyCompletedExchanges` and `verifyRouteStats` methods use Citrus Camel JMX actions to assert how many exchanges a route has processed (and optionally how many failed).

### Test class wiring

Each EIP test class implements `EipTestSupport` and uses the annotations for its target runtime:

On **Quarkus**:

```java
@QuarkusTest
@CitrusSupport
@CitrusConfiguration(
    classes = { KafkaEndpointConfig.class }
)
class EipTests implements EipTestSupport {
    // ...
}
```

On **Spring Boot**:

```java
@SpringBootTest(
    classes = { YourApplication.class }
)
@CitrusSupport
@CitrusConfiguration(
    classes = { KafkaEndpointConfig.class }
)
class EipTests implements EipTestSupport {
    // ...
}
```

The `KafkaEndpointConfig` class provides the Citrus Kafka endpoints used in each test.

### Test dependencies

The common Maven dependencies for all EIP samples:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <!-- or citrus-spring for Spring Boot -->
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-kafka</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-camel</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

All versions are managed by the Citrus BOM — see the sample [pom.xml](https://github.com/citrusframework/citrus-camel-eip-examples/blob/main/pom.xml){:target="_blank"} for the complete setup.

### Running the tests

Execute all tests for a given runtime:

```
mvn verify -pl :quarkus-eip
```

```
mvn verify -pl :spring-boot-eip
```

For YAML DSL tests using Camel JBang:

```
camel test *
```

Run a single YAML test:

```
camel test content-based-router-test.yaml
```

See the individual sample posts for the EIP-specific test logic and scenarios.

  </div>
</article>
