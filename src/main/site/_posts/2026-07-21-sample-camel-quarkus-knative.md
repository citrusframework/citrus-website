---
layout: sample
title: Apache Camel Quarkus - Knative Eventing with CloudEvents
name: camel-knative
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel Knative eventing and CloudEvents in Quarkus using Citrus
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-knative/
---

Knative eventing brings a powerful, event-driven programming model to Kubernetes. An event source detects a change — a file uploaded, a message posted, a database row updated — wraps it in a CloudEvent, and delivers it to a broker. Consumers subscribe to the broker with triggers that filter by event type. It is a clean architecture, but testing it traditionally requires a running Kubernetes cluster with Knative installed, which is a heavyweight dependency for a development feedback loop.

What if you could test the entire flow locally, without Kubernetes, without a real Knative installation, and without touching real AWS services?

This post walks you through testing an Apache Camel Knative event source in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. The application picks up files from an S3 bucket, transforms them into CloudEvents, and sends them to a Knative broker. Citrus provides a local Knative broker, provisions an S3 service through LocalStack, and validates every CloudEvent attribute — all within a standard JUnit 5 test.

By the end you will have a recipe for testing Knative eventing locally with zero cloud infrastructure.

# The application under test

The example application implements a Knative event source. It polls an S3 bucket for new files, converts each file into a CloudEvent, and delivers the event to a Knative broker. In production this would run on a Kubernetes cluster with Knative installed. In tests, everything runs on localhost.

```
Amazon S3 upload --> Camel aws-s3-source Kamelet --> CloudEvent transform --> Knative broker
```

The entire integration logic fits in a single Camel route:

```java
public class Routes extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("kamelet:aws-s3-source")
                .transformDataType(
                    new DataType("http:application-cloudevents"))
                .to("knative:event/org.apache.camel.event.messages" +
                    "?kind=Broker&name=default");
    }
}
```

Three steps, three lines of code. The `aws-s3-source` Kamelet polls the S3 bucket for new objects. The `transformDataType` call converts the exchange into a CloudEvent HTTP message — this is Camel's built-in data type transformation that wraps the payload with the correct `Content-Type` header and CloudEvent metadata structure. Finally, the `knative:event` endpoint delivers the CloudEvent to a Knative broker named `default`.

The Kamelet is a pre-built, reusable Camel connector. It encapsulates all the S3 polling logic — connection management, credential handling, object consumption — behind a simple URI. You configure it through Camel property placeholders, and it works identically against real AWS or a LocalStack emulator.

## Configuring the Knative component

The Camel Knative component needs to know where to send events. In a real Kubernetes deployment, Knative injects the broker URL through a `SinkBinding` that sets the `K_SINK` environment variable. For local development and testing, the component reads a JSON environment file instead.

The application configures this in `application.properties`:

```properties
camel.component.knative.environment-path=classpath:knative.json

camel.component.knative.ceOverride[ce-type]=dev.knative.eventing.aws-s3
camel.component.knative.ceOverride[ce-source]=dev.knative.eventing.aws-s3-source
camel.component.knative.ceOverride[ce-subject]=aws-s3-source
```

The `ceOverride` settings explicitly set the CloudEvent metadata on every produced event. This makes the event look like a proper Knative AWS S3 source event, with a specific type, source, and subject that downstream consumers can use for trigger filtering.

The `knative.json` file defines the broker as a sink resource:

```json
{
  "resources": [
    {
      "name": "default",
      "type": "event",
      "endpointKind": "sink",
      "url": "{% raw %}{{k.sink:http://localhost:8080}}{% endraw %}",
      "objectApiVersion": "eventing.knative.dev/v1",
      "objectKind": "Broker",
      "objectName": "default"
    }
  ]
}
```

The broker URL is resolved from the `k.sink` property, defaulting to `http://localhost:8080`. In production, this property would be set by the Knative SinkBinding. In tests, the application's test profile sets `%test.k.sink=http://localhost:8080`, pointing Camel at the local Citrus Knative broker that we will set up in the test.

# Adding Citrus to the project

This example spans two technologies — Knative eventing and AWS S3 — so the test needs several Citrus modules:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-knative</artifactId>
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

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle, ensuring both frameworks start and stop together.
- **citrus-knative** provides the `knative()` DSL for creating local brokers, sending events, and receiving and validating CloudEvents — all without a Kubernetes cluster.
- **citrus-testcontainers** adds the `@LocalStackContainerSupport` annotation for provisioning a LocalStack container with S3 enabled.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

The key module is **citrus-knative**. It provides a lightweight, local implementation of a Knative broker that runs as an HTTP endpoint. Camel sends CloudEvents to this endpoint just as it would to a real Knative broker on Kubernetes, and Citrus captures every event for validation.

# Understanding the test

The test class combines Citrus Knative support with LocalStack container provisioning to verify the complete event source flow:

```java
@QuarkusTest
@CitrusSupport
@LocalStackContainerSupport(services = AwsService.S3,
        containerLifecycleListener = QuarkusApplicationTest.class)
public class QuarkusApplicationTest
        implements TestActionSupport,
                   ContainerLifecycleListener<LocalStackContainer> {

    @CitrusResource
    GherkinTestActionRunner runner;

    @CitrusResource
    private LocalStackContainer localStackContainer;

    final String s3Key = "message.txt";
    final String s3Data = "Hello Knative!";
    final String s3BucketName = "knative-bucket";
}
```

Three annotations set up the entire test infrastructure:

- `@QuarkusTest` starts the Quarkus application and the Camel route.
- `@CitrusSupport` activates the Citrus framework.
- `@LocalStackContainerSupport` starts a LocalStack container with the S3 service enabled.

The class implements two interfaces. `TestActionSupport` provides access to the `knative()` DSL builder. `ContainerLifecycleListener<LocalStackContainer>` lets the test hook into the container lifecycle to create the S3 bucket and inject connection properties before the application starts.

## Provisioning the S3 bucket

When the LocalStack container starts, the lifecycle listener creates the bucket and returns the Camel Kamelet properties that connect the application to the emulated S3 service:

```java
@Override
public Map<String, String> started(LocalStackContainer container) {
    S3Client s3Client = container.getClient(AwsService.S3);
    s3Client.createBucket(
            builder -> builder.bucket(s3BucketName));

    return Map.of(
        "camel.kamelet.aws-s3-source.accessKey",
            container.getAccessKey(),
        "camel.kamelet.aws-s3-source.secretKey",
            container.getSecretKey(),
        "camel.kamelet.aws-s3-source.region",
            container.getRegion(),
        "camel.kamelet.aws-s3-source.bucketNameOrArn",
            s3BucketName,
        "camel.kamelet.aws-s3-source.uriEndpointOverride",
            container.getServiceEndpoint().toString(),
        "camel.kamelet.aws-s3-source.overrideEndpoint", "true",
        "camel.kamelet.aws-s3-source.forcePathStyle", "true"
    );
}
```

The returned map overrides the Camel Kamelet properties at runtime. The `overrideEndpoint` and `forcePathStyle` options enable LocalStack compatibility — without them, the AWS SDK would try to use virtual-hosted-style bucket addressing, which does not work with a local emulator. All other properties (access key, secret key, region, endpoint URL) point the Kamelet at the LocalStack container instead of real AWS.

This is the same pattern you see in the [Camel S3-to-Kafka sample](/samples/camel-quarkus-aws-s3-kafka/) — the lifecycle listener bridges the gap between dynamically assigned container ports and the application's configuration properties.

## Creating a local Knative broker

The first step in the test creates a local Knative broker:

```java
runner.given(
    knative()
        .brokers()
        .create("default")
        .clusterType(ClusterType.LOCAL)
);
```

`ClusterType.LOCAL` is the critical setting here. It tells Citrus to create a local HTTP endpoint that acts as a Knative broker instead of calling the Kubernetes API to create a real broker resource. The broker listens on `localhost:8080` — the same URL that the Camel Knative component resolves from `k.sink` in the test profile.

From Camel's perspective, this local broker is indistinguishable from a real Knative broker. It accepts HTTP POST requests with CloudEvent headers, stores the events, and makes them available for the test to receive and validate. The difference is entirely in the infrastructure: no Kubernetes, no Istio, no Knative operator.

## Triggering the event source

With the broker and the S3 bucket in place, the test triggers the event source by uploading a file to LocalStack S3:

```java
runner.when(this::uploadS3File);
```

The `uploadS3File` method uses the AWS SDK's S3 client to perform a multipart upload:

```java
private void uploadS3File(TestContext context) {
    S3Client s3Client =
            localStackContainer.getClient(AwsService.S3);

    CreateMultipartUploadResponse initResponse =
            s3Client.createMultipartUpload(
                b -> b.bucket(s3BucketName).key(s3Key));

    String etag = s3Client.uploadPart(
            b -> b.bucket(s3BucketName)
                    .key(s3Key)
                    .uploadId(initResponse.uploadId())
                    .partNumber(1),
            RequestBody.fromString(s3Data)).eTag();

    s3Client.completeMultipartUpload(
            b -> b.bucket(s3BucketName)
                .multipartUpload(CompletedMultipartUpload.builder()
                    .parts(Collections.singletonList(
                        CompletedPart.builder()
                            .partNumber(1)
                            .eTag(etag).build()))
                    .build())
                .key(s3Key)
                .uploadId(initResponse.uploadId()));
}
```

The file `message.txt` with content `Hello Knative!` lands in the `knative-bucket` on LocalStack. The Camel `aws-s3-source` Kamelet polls the bucket, detects the new object, consumes it, and triggers the route. The route transforms the content into a CloudEvent and posts it to the Citrus local broker.

## Receiving and validating CloudEvents

The final step receives the CloudEvent from the local broker and validates both the payload and the CloudEvent metadata attributes:

```java
runner.then(
    knative()
        .event()
        .receive()
        .serviceName("default")
        .eventData(s3Data)
        .attribute("ce-id",
            "@matches([0-9A-Z]{15}-[0-9]{16})@")
        .attribute("ce-type",
            "dev.knative.eventing.aws-s3")
        .attribute("ce-source",
            "dev.knative.eventing.aws-s3-source")
        .attribute("ce-subject", "aws-s3-source")
);
```

The `receive()` action waits for a CloudEvent to arrive on the broker named `default` and validates five things:

- **`eventData`** — the uploaded file content `Hello Knative!`. This confirms that the S3 object content survived the entire pipeline: upload to LocalStack, consumption by the Kamelet, CloudEvent transformation, and delivery to the broker.
- **`ce-id`** — a generated identifier matching the pattern `[0-9A-Z]{15}-[0-9]{16}`. The `@matches()@` syntax is a Citrus validation expression that applies a regular expression, useful when the exact value is dynamic but the format is predictable.
- **`ce-type`** — `dev.knative.eventing.aws-s3`, set by the `ceOverride` configuration. Downstream consumers would use this type in their trigger filters to subscribe only to S3 events.
- **`ce-source`** — `dev.knative.eventing.aws-s3-source`, identifying the origin of the event.
- **`ce-subject`** — `aws-s3-source`, the subject of the event.

These CloudEvent attributes are part of the [CloudEvents specification](https://cloudevents.io/) — a CNCF standard for describing events in a common way. By validating them explicitly, the test ensures that downstream consumers relying on these attributes for routing and filtering will work correctly with the events this source produces.

# How it all fits together

When you run the test with `./mvnw verify`, here is the sequence of events:

1. **`@LocalStackContainerSupport` launches** a LocalStack container with the S3 service and the lifecycle listener creates the `knative-bucket`.
2. **Camel Kamelet properties** are injected from the lifecycle listener, pointing the `aws-s3-source` Kamelet at LocalStack.
3. **Quarkus starts** the application in test mode. The Camel route begins polling the S3 bucket.
4. **Citrus creates** a local Knative broker named `default` on `localhost:8080`.
5. **The test uploads** `message.txt` with content `Hello Knative!` to the S3 bucket via the AWS SDK.
6. **The Kamelet detects** the new object and consumes it from the bucket.
7. **The route transforms** the content into a CloudEvent with the configured `ce-type`, `ce-source`, and `ce-subject` attributes.
8. **Camel posts** the CloudEvent to the Citrus local broker at `localhost:8080`.
9. **Citrus receives** the event and validates the payload and all CloudEvent attributes.
10. **The LocalStack container** is stopped and cleaned up automatically.

The test runs entirely on localhost. No Kubernetes cluster, no Knative installation, no AWS account. Yet it exercises the complete event source flow — from S3 upload to CloudEvent delivery — with real container-based infrastructure for S3 and a faithful local broker implementation for Knative.

# Why local Knative testing matters

Knative eventing introduces a level of indirection that can be hard to test. Your application produces events and sends them to a broker, but you typically have no visibility into what the broker receives until a consumer processes the event downstream. Bugs in CloudEvent attribute configuration, payload transformation, or broker connectivity only surface in a deployed environment — which is slow, expensive, and hard to debug.

Citrus eliminates this gap. The local Knative broker captures every event exactly as the real broker would receive it, and the test can inspect every attribute. If your `ce-type` is wrong, the test fails immediately with a clear mismatch message. If the payload transformation drops content, you see it in the diff. You do not need to deploy, wait for a consumer to pick up the event, and dig through logs to find the problem.

This is especially valuable during development. When you are iterating on the CloudEvent format or adding new event sources, a local test that runs in seconds provides a much tighter feedback loop than deploying to a staging cluster. The test acts as a contract — it defines what the event source promises to deliver, and it verifies that promise on every build.

# Where to go from here

This example covers a single event source with one event type, but the same pattern extends to more complex eventing scenarios:

- **Multiple event types**: Add a second route that produces events with a different `ce-type` and verify that the broker receives both types with the correct attributes.
- **Trigger filtering**: Create multiple local brokers or use trigger filters to verify that events are routed to the correct consumer based on CloudEvent attributes.
- **Event chains**: Test a pipeline where one event triggers processing that produces a second event. Citrus can validate both the intermediate and final events.
- **Error handling**: Verify that the event source handles S3 connectivity issues gracefully — does it retry? Does it produce a dead-letter event?
- **SQS and SNS sources**: Apply the same LocalStack pattern with `AwsService.SQS` or `AwsService.SNS` to test event sources backed by other AWS services.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers Knative support, Testcontainers integration, and the many other transports that Citrus supports.
