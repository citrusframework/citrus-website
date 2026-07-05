---
layout: sample
title: Apache Camel Quarkus - AWS S3 to Kafka Pipeline
name: camel-aws-s3-kafka
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel S3-to-Kafka routes in Quarkus with Citrus, LocalStack, and Testcontainers
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-aws-s3-kafka/
---

Cloud-native integration often starts with a file landing in an S3 bucket. A new CSV is uploaded, a data export arrives, a batch of events is dropped — and your application needs to pick it up, process the content, and publish the results to a Kafka topic for downstream consumption. Apache Camel handles this pattern elegantly with its S3 Kamelet and Splitter EIP.

Testing it is the challenge. You need an S3-compatible object store, a Kafka broker, a bucket created before the test runs, and a way to upload files and verify the downstream output — all coordinated in a single test. You certainly do not want to test against real AWS services.

This post walks you through testing an Apache Camel route that bridges AWS S3 and Kafka in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus provisions a LocalStack container for S3 emulation, uploads files through Camel's S3 component, and validates that each line of the file arrives as a separate JSON event on a Kafka topic.

By the end you will have a recipe for testing cloud-native Camel pipelines without touching real AWS services.

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
        from("kamelet:aws-s3-source?" +
                "bucketNameOrArn={{aws.s3.bucketNameOrArn}}&" +
                "region={{aws.s3.region}}&" +
                "overrideEndpoint=true&" +
                "forcePathStyle=true&" +
                "uriEndpointOverride={{aws.s3.uriEndpointOverride}}&" +
                "accessKey={{aws.s3.accessKey}}&" +
                "secretKey={{aws.s3.secretKey}}")
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

The route uses the `aws-s3-source` **Kamelet** — a pre-built, reusable Camel connector — to poll the S3 bucket for new objects. The `overrideEndpoint` and `forcePathStyle` options enable LocalStack compatibility, so the same route works against both real AWS and a local emulator. All connection properties are externalized as Camel property placeholders, resolved from `application.properties`.

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
            String serviceEndpoint = container.getEndpointOverride(AwsService.S3)
                    .toString();

            S3Client s3Client = container.getClient(AwsService.S3);
            s3Client.createBucket(builder -> builder.bucket(BUCKET_NAME));

            return Map.of(
                    "aws.s3.bucketNameOrArn", BUCKET_NAME,
                    "aws.s3.uriEndpointOverride", serviceEndpoint,
                    "aws.s3.accessKey", container.getAccessKey(),
                    "aws.s3.secretKey", container.getSecretKey(),
                    "aws.s3.region", container.getRegion()
            );
        }
    }
}
```

The `services = AwsService.S3` parameter tells LocalStack to start only the S3 service, keeping the container lightweight. The lifecycle listener does two things: it creates the S3 bucket that the Camel route expects, and it returns a map of connection properties that override the `application.properties` defaults.

These returned properties — bucket name, endpoint URL, access key, secret key, region — are automatically injected into the Quarkus application context. When the Camel route resolves its `aws.s3.*` placeholders, they point to the LocalStack container instead of real AWS.

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

# Uploading files through Camel's S3 component

The test uses Camel's `aws2-s3` component to upload a multi-line file to the LocalStack bucket. As with the [MQTT sample](/samples/camel-quarkus-mqtt/), Citrus leverages Camel's component library for protocols it does not natively support:

```java
runner.when(
    camel()
        .send()
        .endpoint("aws2-s3://citrus-camel-demo?" +
                "overrideEndpoint=true&" +
                "forcePathStyle=true&" +
                "uriEndpointOverride={{aws.s3.uriEndpointOverride}}&" +
                "accessKey={{aws.s3.accessKey}}&" +
                "secretKey={{aws.s3.secretKey}}&" +
                "region={{aws.s3.region}}")
        .message()
        .fork(true)
        .body("Hello Camel!\nHello Citrus!\nHello Quarkus!")
        .header("CamelAwsS3Key", "hello.txt")
);
```

The `camel().send()` DSL sends a message through Camel's `aws2-s3` endpoint, which uploads the content as an S3 object. The `CamelAwsS3Key` header sets the object key to `hello.txt`. The property placeholders resolve to the same LocalStack connection settings that the application route uses.

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
3. **Quarkus starts** the application in test mode with S3 and Kafka connection properties pointing to the test containers.
4. **Apache Camel discovers** the route and starts polling the S3 bucket via the `aws-s3-source` Kamelet.
5. **The test uploads** `hello.txt` with three lines to the S3 bucket through Camel's `aws2-s3` component.
6. **The Kamelet detects** the new object and consumes it from the bucket.
7. **The Splitter EIP** breaks the file into three messages — one per line.
8. **The filter** removes any empty lines.
9. **Each line is wrapped** in a JSON structure: `{ "message": "Hello Camel!" }`.
10. **Three Kafka messages** are published to the `s3-events` topic.
11. **Citrus receives** each message and validates the JSON body.
12. **Both containers** are stopped and cleaned up automatically.

The critical point is that the same connection properties drive both the application and the test. The `LocalStackConfigurer` returns properties like `aws.s3.uriEndpointOverride` that the Camel route resolves through its placeholders. The test's `aws2-s3` endpoint uses the same placeholders. This ensures that both sides always point to the same LocalStack instance.

There is one piece of test configuration worth mentioning:

```properties
quarkus.arc.ignored-split-packages=org.citrusframework.*
```

This avoids CDI bean discovery conflicts between Citrus and Quarkus and is a one-line addition you set once and forget.

# LocalStack — real AWS behavior without real AWS

LocalStack emulates AWS services locally with high fidelity. For S3 in particular, it supports bucket creation, object upload and download, listing, and deletion — everything the Camel `aws-s3-source` Kamelet needs.

The `@LocalStackContainerSupport` annotation makes this integration seamless. You specify which AWS services you need (`AwsService.S3`, `AwsService.SQS`, `AwsService.SNS`, etc.), and the annotation starts a container with only those services enabled. The lifecycle listener gives you a typed `LocalStackContainer` instance with convenience methods like `getClient(AwsService.S3)` for creating AWS SDK clients and `getEndpointOverride()` for retrieving the service URL.

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
