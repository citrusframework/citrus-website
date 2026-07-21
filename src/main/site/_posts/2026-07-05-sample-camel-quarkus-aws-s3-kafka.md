---
layout: sample
title: Apache Camel Quarkus - AWS S3 to Kafka Pipeline
name: camel-aws-s3-kafka
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel S3-to-Kafka routes in Quarkus with Citrus, LocalStack, Kamelet property binding, and Testcontainers
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-aws-s3-kafka/
---

Cloud-native integration often starts with a file landing in an S3 bucket. A new CSV is uploaded, a data export arrives, a batch of events is dropped — and your application needs to pick it up, process the content, and publish the results to a Kafka topic for downstream consumption. Apache Camel handles this pattern elegantly with its S3 Kamelet and Splitter EIP.

Testing it is the challenge. You need an S3-compatible object store, a Kafka broker, a bucket created before the test runs, and a way to upload files and verify the downstream output — all coordinated in a single test. You certainly do not want to test against real AWS services.

This post walks you through testing an Apache Camel route that bridges AWS S3 and Kafka in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus provisions a LocalStack container for S3 emulation, uploads files through Camel's type-safe S3 endpoint DSL, and validates that each line of the file arrives as a separate JSON event on a Kafka topic.

By the end you will have a recipe for testing cloud-native Camel pipelines without touching real AWS services — and a clean separation between production and test configuration.

# The application under test

The example application implements a file ingestion pipeline. It polls an S3 bucket for new files, splits each file into individual lines, wraps every line in a JSON structure, and publishes the resulting events to a Kafka topic.

```
S3 Bucket (citrus-camel-demo) --> Split by newline --> Filter empty lines --> JSON wrap --> Kafka Topic (s3-events)
```

The entire logic lives in a single Camel route:

```java
public class Routes extends EndpointRouteBuilder {

    @Override
    public void configure() throws Exception {
        from("kamelet:aws-s3-source")
            .split(body().tokenize("\n"))
            .filter(simple("${body} != \"\""))
            .setBody()
                .simple("""
                    { "message": "${body}" }
                    """)
            .to(kafka("s3-events"));
    }
}
```

The route uses the `aws-s3-source` **Kamelet** — a pre-built, reusable Camel connector — to poll the S3 bucket for new objects. Notice that the Kamelet URI carries no inline connection parameters. All `aws-s3-source` properties are bound through Camel's standard **Kamelet property binding** mechanism in `application.properties`:

```properties
# Kamelet source properties
camel.kamelet.aws-s3-source.bucketNameOrArn=citrus-camel-demo
camel.kamelet.aws-s3-source.region=us-east-1
camel.kamelet.aws-s3-source.accessKey=accesskey
camel.kamelet.aws-s3-source.secretKey=secretkey

# Kafka endpoint
kafka.bootstrap.servers=my-cluster.namespace.local
%test.kafka.bootstrap.servers=localhost:9092
```

This is the idiomatic Camel Kamelet approach and keeps the route URI clean. Test-specific settings like `overrideEndpoint`, `forcePathStyle`, and `uriEndpointOverride` — which only make sense against a local container — are **not** present here. They are injected dynamically by the test's `LocalStackConfigurer` lifecycle listener (see below), so the production configuration remains unaffected.

The `%test.kafka.bootstrap.servers` entry is a Quarkus profile-scoped override that applies only in the `test` profile, keeping the production value clean while wiring the Kafka client to the Testcontainer-managed broker during tests.

Once a file arrives, the **Splitter EIP** breaks the content on newline characters. A file containing three lines becomes three separate Camel exchanges, each carrying one line as its body. The filter step removes empty lines that might result from trailing newlines.

Each surviving line is wrapped in a JSON structure using a Simple expression: `{ "message": "Hello Camel!" }`. Finally, the JSON event is published to the `s3-events` Kafka topic. A three-line file produces three separate Kafka messages.

# Adding Citrus to the project

This example spans two cloud services — S3 and Kafka — so it needs a few more Citrus modules:

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
- **citrus-camel** provides the `camel()` DSL that lets you use Camel's `aws2-s3` component to upload files to the LocalStack S3 bucket from within the test.
- **citrus-kafka** provides Kafka endpoint implementations for receiving and validating the output events.
- **citrus-testcontainers** adds the `@LocalStackContainerSupport` and `@KafkaContainerSupport` annotations for provisioning both containers.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

The key module here is **citrus-testcontainers**. It provides built-in support for LocalStack — the same project that many teams use for local AWS development — wrapped in a simple annotation that handles container lifecycle, service selection, and property injection.

# Provisioning the test infrastructure

This test needs two containers: a LocalStack instance for S3 and a Kafka broker for the output. Citrus provisions both with annotations.

## The LocalStack S3 service

The `@LocalStackContainerSupport` annotation starts a LocalStack container with only the S3 service enabled:

```java
@LocalStackContainerSupport(services = AwsService.S3,
        containerLifecycleListener = QuarkusApplicationTest.LocalStackConfigurer.class)
class QuarkusApplicationTest {

    public static class LocalStackConfigurer
            implements ContainerLifecycleListener<LocalStackContainer> {
        @Override
        public Map<String, String> started(LocalStackContainer container) {
            S3Client s3Client = container.getClient(AwsService.S3);
            s3Client.createBucket(builder -> builder.bucket(BUCKET_NAME));

            String serviceEndpoint = container.getServiceEndpoint().toString();

            return Map.of(
                    "camel.kamelet.aws-s3-source.bucketNameOrArn", BUCKET_NAME,
                    "camel.kamelet.aws-s3-source.uriEndpointOverride", serviceEndpoint,
                    "camel.kamelet.aws-s3-source.accessKey", container.getAccessKey(),
                    "camel.kamelet.aws-s3-source.secretKey", container.getSecretKey(),
                    "camel.kamelet.aws-s3-source.region", container.getRegion(),
                    "camel.kamelet.aws-s3-source.overrideEndpoint", "true",
                    "camel.kamelet.aws-s3-source.forcePathStyle", "true"
            );
        }
    }
}
```

The `services = AwsService.S3` parameter tells LocalStack to start only the S3 service, keeping the container lightweight. The lifecycle listener does two things: it creates the S3 bucket that the Camel route expects, and it returns a map of Kamelet properties that override the production defaults in `application.properties`.

The returned properties use the `camel.kamelet.aws-s3-source.*` namespace — the same Kamelet property binding mechanism that the production configuration uses. This means the listener can both override existing properties (bucket name, credentials, region) and add test-only settings (`overrideEndpoint`, `forcePathStyle`, `uriEndpointOverride`) that enable LocalStack compatibility. These test-specific settings never appear in the production `application.properties` — they are injected dynamically and only take effect when the test runs.

## The Kafka broker

The Kafka side uses the familiar `@KafkaContainerSupport` annotation with a lifecycle listener that creates the topic:

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
                    new NewTopic("s3-events", 1, (short) 1)
                )).all().get();
            }

            return Collections.emptyMap();
        }
    }
}
```

This is the same pattern you see in the [Camel Kafka sample](/samples/camel-quarkus-kafka/) — the listener creates the `s3-events` topic before the application starts.

# Uploading files through Camel's S3 endpoint DSL

The test uses Camel's `aws2-s3` component to upload a multi-line file to the LocalStack bucket. As with the [MQTT sample](/samples/camel-quarkus-mqtt/), Citrus leverages Camel's component library for protocols it does not natively support. Instead of building a raw URI string with connection parameters, the test binds the `S3Client` from the LocalStack container into the Camel registry and uses the type-safe `aws2S3()` endpoint DSL:

```java
runner.given(
    camel().bind("s3Client", localStackContainer.getClient(AwsService.S3))
);

runner.when(
    camel()
        .send()
        .endpoint(aws2S3(BUCKET_NAME)
                .advanced()
                .amazonS3Client("#s3Client")::getRawUri)
        .message()
        .fork(true)
        .body("Hello Camel!\nHello Citrus!\nHello Quarkus!")
        .header("CamelAwsS3Key", "hello.txt")
);
```

The `camel().bind()` call registers the `S3Client` obtained from the LocalStack container in the Camel registry under the name `s3Client`. The `aws2S3()` endpoint DSL then references this bound client via `#s3Client`, which means the test reuses the same authenticated client that LocalStack already configured — no need to manually construct a URI with access keys, endpoints, and region parameters.

The `fork(true)` option sends the upload asynchronously. The S3 Kamelet polls the bucket at intervals, so there is a delay between the upload and the route picking up the file. Forking lets the test proceed to the verification step while the upload and polling happen in the background.

The file body contains three lines separated by newlines. Once the Camel route picks up this file, the Splitter EIP will produce three separate messages — one for each line.

# Verifying the Kafka events

The output side uses standard Citrus Kafka endpoints:

```java
@CitrusEndpoint
@KafkaEndpointConfig(topic = "s3-events", consumerGroup = "citrus-consumer-1")
KafkaEndpoint s3Events;
```

After uploading the file, the test verifies that each line produces a separate JSON event on the `s3-events` Kafka topic:

```java
runner.then(
    receive()
        .endpoint(s3Events)
        .message()
        .body("{ \"message\": \"Hello Camel!\" }")
);

runner.then(
    receive()
        .endpoint(s3Events)
        .message()
        .body("{ \"message\": \"Hello Citrus!\" }")
);

runner.then(
    receive()
        .endpoint(s3Events)
        .message()
        .body("{ \"message\": \"Hello Quarkus!\" }")
);
```

Three `receive()` actions validate three Kafka messages — one for each line of the uploaded file. Each assertion checks that the JSON wrapper was applied correctly: the line `Hello Camel!` becomes `{ "message": "Hello Camel!" }`. If the Splitter produces the wrong number of messages, or if the JSON transformation is incorrect, Citrus fails the test with a clear message.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events:

1. **`@LocalStackContainerSupport` launches** a LocalStack container with the S3 service and creates the `citrus-camel-demo` bucket.
2. **`@KafkaContainerSupport` launches** a Kafka broker and creates the `s3-events` topic.
3. **Quarkus starts** the application in test mode with Kamelet and Kafka connection properties pointing to the test containers.
4. **Apache Camel discovers** the route and starts polling the S3 bucket via the `aws-s3-source` Kamelet, whose properties are resolved through the `camel.kamelet.aws-s3-source.*` namespace.
5. **The test binds** the LocalStack `S3Client` into the Camel registry and **uploads** `hello.txt` with three lines to the S3 bucket through Camel's type-safe `aws2S3()` endpoint DSL.
6. **The Kamelet detects** the new object and consumes it from the bucket.
7. **The Splitter EIP** breaks the file into three messages — one per line.
8. **The filter** removes any empty lines.
9. **Each line is wrapped** in a JSON structure: `{ "message": "Hello Camel!" }`.
10. **Three Kafka messages** are published to the `s3-events` topic.
11. **Citrus receives** each message and validates the JSON body.
12. **Both containers** are stopped and cleaned up automatically.

The critical point is the clean separation between production and test configuration. The `LocalStackConfigurer` returns `camel.kamelet.aws-s3-source.*` properties that override the production defaults and add test-only settings like `overrideEndpoint` and `forcePathStyle`. The test's `aws2-s3` endpoint reuses the same LocalStack `S3Client` through a Camel registry binding. Both sides point to the same LocalStack instance without polluting the production `application.properties`.

There is one piece of test configuration worth mentioning:

```properties
quarkus.arc.ignored-split-packages=org.citrusframework.*
```

This avoids CDI bean discovery conflicts between Citrus and Quarkus and is a one-line addition you set once and forget.

# LocalStack — real AWS behavior without real AWS

LocalStack emulates AWS services locally with high fidelity. For S3 in particular, it supports bucket creation, object upload and download, listing, and deletion — everything the Camel `aws-s3-source` Kamelet needs.

The `@LocalStackContainerSupport` annotation makes this integration seamless. You specify which AWS services you need (`AwsService.S3`, `AwsService.SQS`, `AwsService.SNS`, etc.), and the annotation starts a container with only those services enabled. The lifecycle listener gives you a typed `LocalStackContainer` instance with convenience methods like `getClient(AwsService.S3)` for creating AWS SDK clients and `getServiceEndpoint()` for retrieving the service URL.

Combined with Camel's Kamelet property binding, this creates a clean layering: production `application.properties` declares the real AWS connection settings, while the `ContainerLifecycleListener` overrides them with LocalStack-specific values and adds test-only options — all through the same `camel.kamelet.*` namespace.

This approach extends to any AWS service that LocalStack supports. If your Camel route consumes from SQS, writes to DynamoDB, or publishes to SNS, the same annotation-and-listener pattern provisions the infrastructure and injects the connection properties.

# Where to go from here

This example covers the fundamentals of S3-to-Kafka testing, but there are several directions you can take it:

- **JSON validation**: Add the `citrus-validation-json` module to validate JSON payloads with JsonPath expressions, ignore patterns, and flexible element ordering instead of exact string comparison.
- **Multiple files**: Upload several files to the bucket and verify that each is processed independently, with the correct number of Kafka events per file.
- **Large files**: Test with files containing hundreds of lines to verify that the Splitter and Kafka producer handle volume correctly.
- **Error scenarios**: Upload a file with malformed content and verify that the route handles it gracefully — does it skip bad lines, route them to a dead-letter topic, or log an error?
- **Other AWS services**: If your Camel route consumes from SQS or publishes to SNS, apply the same `@LocalStackContainerSupport` pattern with the appropriate `AwsService` enum values.
- **Simpler pipelines**: If your Camel route uses Kafka without S3, the [Camel Kafka sample](/samples/camel-quarkus-kafka/) demonstrates a simpler setup. For file-based input without cloud storage, the [file processing pipeline sample](/samples/camel-quarkus-file-inbox/) shows local file consumption.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers Testcontainers integration, Kafka endpoints, Apache Camel support, and the many other transports that Citrus supports.
