---
layout: post
title: Testing Knative Eventing with Citrus — from local brokers to TLS-secured CloudEvents
short-title: Knative Eventing Testing
author: Christoph Deppisch
github: christophd
categories: [blog]
---

Event-driven architecture is everywhere. A file lands in a storage bucket, a database row changes, a sensor publishes a reading — and somewhere downstream, a consumer reacts. [Knative Eventing](https://knative.dev/docs/eventing/) brings this model to Kubernetes with a clean abstraction: event sources produce [CloudEvents](https://cloudevents.io/), brokers route them, triggers filter them, and services consume them. The programming model is elegant, but testing it is not.

![Knative eventing testing](/img/assets/knative-eventing/featured.png)
*AI generated with Google Gemini*

The usual testing approach is to deploy the application to a Kubernetes cluster that has Knative already installed, trigger the event source manually, and check the downstream consumers through event logs or manual inspection. This is slow, expensive, and fragile. It ties your development feedback loop to cluster availability, makes debugging difficult, and creates test environments that are hard to reproduce. What you really want is to test Knative eventing logic locally, with real message validation, but without any heavy cloud infrastructure.

This is exactly what the [Citrus](https://citrusframework.org) `citrus-knative` module provides. It gives you a purpose-built DSL for creating local Knative brokers, producing and consuming CloudEvents, managing triggers with attribute filters, and validating every CloudEvent attribute — all within a standard JUnit test support. Citrus also supports testing against real Kubernetes clusters when you need to verify behavior in a deployed environment, but this is for a different day.

This post explores the full breadth of Citrus Knative testing capabilities. We start with the local broker abstraction that eliminates the need for a Kubernetes cluster, walk through CloudEvent production and consumption, cover trigger-based event filtering, and finish with TLS-secured eventing. Each capability is illustrated with code from working sample projects that running Knative eventing applications with Quarkus.  You can clone and run the code samples yourself without any Cloud infrastructure involved.

# The Citrus Knative module

Add the module as a test dependency to get started:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-knative</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

The module provides three groups of test actions, each accessible through the `knative()` DSL:

**Broker management** — create, verify, and delete Knative brokers. In local mode, Citrus starts a lightweight HTTP server that acts as a broker. In Kubernetes mode, Citrus creates real broker resources on the cluster through the Fabric8 Knative client.

**Event operations** — send CloudEvents to a broker and receive CloudEvents from a service. The DSL lets you set and validate all CloudEvent attributes (`ce-type`, `ce-source`, `ce-subject`, `ce-id`, `ce-specversion`) alongside the event payload.

**Trigger and channel management** — create triggers with attribute filters that route events from brokers to services or channels. Create channels and subscriptions for pub/sub event distribution.

All resources created during a test are automatically cleaned up when the test finishes — even if the test fails. This default behavior prevents resource leaks and keeps test environments clean. You can disable auto-removal with the system property `citrus.knative.auto.remove.resources=false` when you want resources to persist across tests.

# Local mode Knative brokers

The single most powerful feature of the `citrus-knative` module is the ability to create local Knative brokers that run as plain HTTP endpoints on your localhost, with no Kubernetes cluster required. This is the foundation that makes all other Knative testing capabilities practical for local development.

Creating a local broker is a single test action:

```java
runner.given(
    knative()
        .brokers()
        .create("default")
        .clusterType(ClusterType.LOCAL)
);
```

`ClusterType.LOCAL` is the critical setting. It tells Citrus to start a local HTTP server that accepts CloudEvent POST requests, stores the events, and makes them available for the test to receive and validate. From the application's perspective, this server is indistinguishable from a real Knative broker — it speaks the same HTTP-based CloudEvent protocol.

The broker listens on `localhost:8080` by default. Your application just needs to point its Knative sink URL at this address. For a Camel Quarkus application, this is typically a one-line property override in the test profile:

```properties
k.sink=http://localhost:8080
```

In a real Kubernetes deployment, Knative injects the broker URL through a `SinkBinding` that sets the `K_SINK` environment variable. The test profile property achieves the same effect without any Kubernetes infrastructure.

A pro tip is to use the Quarkus test profile in the application.properties so the URL is only used during testing.

```properties
%test.k.sink=http://localhost:8080
```

What makes this approach valuable is fidelity. The local broker does not mock the Knative API — it implements the same HTTP CloudEvent ingestion that a real broker uses. If your application constructs CloudEvents incorrectly — wrong headers, malformed payload, missing required attributes — the test catches it. The difference is entirely in the infrastructure layer: no Kubernetes, no Istio, no Knative operator.

You can also verify that a broker is up and ready before proceeding with the test:

```java
runner.given(
    knative()
        .client(knativeClient)
        .brokers()
        .verify("my-broker")
        .inNamespace("my-namespace")
);
```

This resolves the Knative broker named `my-broker` and checks the running/ready status. This is more useful in Kubernetes mode, where broker creation is asynchronous, and you might want to wait for the broker to become ready before sending events.

Depending on the cluster mode `local` or `kubernetes` the verify test action looks for a local Http Knative broker service or connects to the Kubernetes namespace to inspect the broker status.

# Producing CloudEvents

Once a broker is available, Citrus can produce CloudEvents and send them to the broker. This is useful when you need to simulate upstream event sources — for example, testing how a consumer service handles events with specific attributes or payloads.

The `send()` action constructs a complete CloudEvent with all required attributes:

```java
runner.when(
    knative()
        .event()
        .send()
        .brokerUrl("http://localhost:8080")
        .attribute("ce-type", "dev.knative.eventing.order")
        .attribute("ce-source", "order-service")
        .eventData("""
            { "orderId": "12345", "status": "confirmed" }
        """)
);
```

Under the hood, Citrus translates this into an HTTP POST request to the broker with proper CloudEvent headers:

```
POST http://localhost:8080
Content-Type: application/json
Ce-Id: 2818d613-bc75-4b25-b570-b825bbe33378
Ce-Type: dev.knative.eventing.order
Ce-Specversion: 1.0
Ce-Source: order-service

{ "orderId": "12345", "status": "confirmed" }
```

If you do not set the `ce-id` or `ce-specversion` attributes explicitly, Citrus generates sensible defaults — a UUID for the ID and `1.0` for the spec version. You can override any attribute through the fluent `.attribute()` calls.

The send action supports both inline and pre-defined event data. For larger payloads, you can load the event data from a file resource in a separate step before the send action, keeping the action itself focused on the CloudEvent attributes.

# Receiving and validating CloudEvents

Receiving events is the other side of the conversation. A Citrus test receives CloudEvents that were delivered to a service by the Knative broker (or directly by an application), then validates the payload and attributes against expected values.

The `receive()` action waits for an event to arrive and validates its content:

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

Several things are worth noting here. The `serviceName` identifies which broker or service to receive from — in local mode, this matches the broker name used in the `create()` action. The `eventData` assertion validates the payload content. Each `.attribute()` call validates a specific CloudEvent header.

The `@matches(...)@` syntax is a Citrus validation expression that applies a regular expression to the attribute value. This is essential for dynamic attributes like `ce-id` where the exact value changes on every run but the format is predictable. Citrus supports a rich set of validation matchers including `@startsWith()@`, `@contains()@`, `@isNumber()@`, and many others that help you write precise yet flexible assertions.

This receive-and-validate pattern is what gives Knative eventing testing its teeth. In a deployed Knative environment, the events flowing through the broker are invisible — you only see them indirectly when a consumer processes them. With Citrus, you validate the events at the broker level, catching problems in CloudEvent construction, attribute configuration, and payload transformation before they propagate to downstream consumers.

## A real-world example

The [Knative eventing sample](/samples/camel-quarkus-knative/) demonstrates this pattern end-to-end. The application under test is a Camel Quarkus event source that polls an S3 bucket for new files, transforms each file into a CloudEvent, and delivers it to a Knative broker:

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

The test creates a local Knative broker, provisions an S3 bucket through LocalStack, uploads a file, and validates the resulting CloudEvent:

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

    final String s3Data = "Hello Knative!";

    @Test
    void shouldProduceCloudEvents() {
        runner.given(
            knative()
                .brokers()
                .create("default")
                .clusterType(ClusterType.LOCAL)
        );

        runner.when(this::uploadS3File);

        runner.then(
            knative()
                .event()
                .receive()
                .serviceName("default")
                .eventData(s3Data)
                .attribute("ce-type",
                    "dev.knative.eventing.aws-s3")
                .attribute("ce-source",
                    "dev.knative.eventing.aws-s3-source")
                .attribute("ce-subject", "aws-s3-source")
        );
    }
}
```

The Given-When-Then structure makes the test's intent immediately clear: given a local Knative broker, when a file is uploaded to S3, then a CloudEvent arrives at the broker with the expected payload and attributes. No Kubernetes cluster, no real AWS account, no manual log inspection.

# SSL/TLS secured eventing

Production Knative deployments almost always secure broker communication with TLS. Event sources connect over HTTPS, present client certificates, and validate the broker's identity through a trust chain. Testing SSL is important because SSL configuration is one of the most common sources of deployment failures — a missing truststore entry, an expired certificate, or a wrong keystore password can all cause silent failures.

Citrus supports TLS-secured Knative testing through its HTTP server infrastructure. Instead of using the high-level `knative()` broker DSL, you configure an explicit HTTP server with SSL settings:

```java
@BindToRegistry
public HttpServer knativeBroker = HttpEndpoints.http()
        .server()
        .port(8080)
        .timeout(5000L)
        .securePort(8443)
        .secured(HttpSecureConnection.ssl()
                .keyStore("classpath:keystore/server.jks",
                          "secr3t")
                .trustStore("classpath:keystore/truststore.jks",
                            "secr3t"))
        .autoStart(true)
        .build();
```

This creates a Citrus HTTP server that listens on both port 8080 (plain HTTP) and port 8443 (HTTPS). The `secured()` call configures TLS with a server keystore and a truststore. The server presents the certificate from `server.jks` during the TLS handshake, and uses `truststore.jks` to verify client certificates for mutual TLS.

The test then validates CloudEvents using the `http().server()` DSL, which gives explicit control over the HTTP exchange:

```java
runner.then(
    http().server(knativeBroker)
            .receive()
            .post()
            .message()
            .body(s3Data)
            .header("ce-type",
                "dev.knative.eventing.aws-s3")
            .header("ce-source",
                "dev.knative.eventing.aws-s3-source")
            .header("ce-subject", "aws-s3-source")
);

runner.then(
    http().server(knativeBroker)
            .send()
            .response(HttpStatus.OK)
);
```

The assertions are the same — you validate CloudEvent headers and the payload body. The difference is that the exchange happens over a TLS-encrypted channel. If the SSL handshake fails — wrong certificate, expired CA, mismatched hostname — the test fails before the receive action even executes, giving you immediate feedback about SSL configuration problems.

For the application side, SSL is configured through properties that the `KnativeSslClientOptions` CDI bean reads at startup:

```java
return Map.ofEntries(
    Map.entry("camel.knative.client.ssl.enabled", "true"),
    Map.entry("camel.knative.client.ssl.verify.hostname",
              "false"),
    Map.entry("camel.knative.client.ssl.key.path",
              "keystore/client.pem"),
    Map.entry("camel.knative.client.ssl.key.cert.path",
              "keystore/client.crt"),
    Map.entry("camel.knative.client.ssl.truststore.path",
              "keystore/truststore.jks"),
    Map.entry("camel.knative.client.ssl.truststore.password",
              "secr3t")
);
```

The test controls both sides of the TLS connection — it provides the server certificate through the HTTP server configuration and the client certificate through these properties — so the handshake succeeds with self-signed test certificates and no external PKI infrastructure.

The [Knative SSL sample](/samples/camel-quarkus-knative-ssl/) demonstrates this pattern end-to-end with mutual TLS between a Camel Quarkus application and a Citrus HTTPS broker.

# Testing against real Kubernetes clusters

While local testing covers the majority of development scenarios, sometimes you need to verify behavior on a real Kubernetes cluster with Knative installed. The `citrus-knative` module supports this through the Fabric8 Knative client.

Configure the Kubernetes and Knative clients as Citrus beans:

```java
@BindToRegistry
public KubernetesClient kubernetesClient() {
    return new KubernetesClientBuilder().build();
}

@BindToRegistry
public KnativeClient knativeClient() {
    return kubernetesClient().adapt(KnativeClient.class);
}
```

With these clients registered, the same DSL actions you used in local mode now operate against the real cluster. Create a broker without `ClusterType.LOCAL`:

```java
runner.given(
    knative()
        .client(knativeClient)
        .brokers()
        .create("my-broker")
        .inNamespace("my-namespace")
);
```

This creates an actual Knative Broker resource on the cluster. The broker goes through the normal Knative reconciliation process — the Knative operator provisions the underlying channel, the ingress, and the filter services. You can verify readiness before proceeding:

```java
runner.given(
    knative()
        .client(knativeClient)
        .brokers()
        .verify("my-broker")
        .inNamespace("my-namespace")
);
```

Sending events targets the real broker URL:

```java
runner.when(
    knative()
        .event()
        .send()
        .brokerUrl("http://broker-ingress.knative-eventing" +
                   ".svc.cluster.local/my-namespace/my-broker")
        .attribute("ce-type", "greeting")
        .eventData("{ \"msg\": \"Hello Knative!\" }")
);
```

And receiving events uses a real Kubernetes service that the test creates through the Citrus Kubernetes module. The Citrus HTTP server backing the service receives events dispatched by the broker through real Knative triggers, and the test validates them with the same assertions used in local mode.

This dual-mode capability is one of the strengths of the `citrus-knative` module. You develop and iterate locally with `ClusterType.LOCAL`, then run the same test logic against a real cluster in CI/CD by switching to the Kubernetes client. The assertions stay the same — only the infrastructure changes.

# Triggers and event filtering

In Knative, triggers control which events reach which consumers. A trigger subscribes to a broker and forwards matching events to a service, filtering on CloudEvent attributes like `type`, `source`, or custom extensions. Testing trigger behavior is important because a misconfigured filter can silently drop events or route them to the wrong consumer.

Citrus can create trigger resources as part of the test:

```java
runner.given(
    knative()
        .client(knativeClient)
        .trigger()
        .create("order-trigger")
        .broker("my-broker")
        .service("order-processor")
        .filter("type", "dev.knative.eventing.order")
        .inNamespace("my-namespace")
);
```

This creates a Knative Trigger resource that watches the `my-broker` broker for events with `type=dev.knative.eventing.order` and forwards them to the `order-processor` service. The resulting Trigger resource looks like this on the cluster:

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: order-trigger
spec:
  broker: my-broker
  filter:
    attributes:
      type: dev.knative.eventing.order
  subscriber:
    ref:
      apiVersion: v1
      kind: Service
      name: order-processor
```

You can add multiple filter attributes to narrow the event selection further. The trigger only dispatches events that match all specified attributes — this is Knative's AND-based filtering model.

To consume the filtered events, you first need a Kubernetes service. Citrus integrates with its Kubernetes support module to create a service that routes to a local HTTP server:

```java
runner.given(
    kubernetes()
        .client(k8sClient)
        .service()
        .create("order-processor")
        .portMapping(80, 8080)
        .inNamespace("my-namespace")
);
```

Citrus automatically starts a local HTTP server on port 8080 that receives events dispatched by the trigger. You can then receive and validate the events in the test.

Triggers also support forwarding events to channels instead of services, which is useful for fan-out scenarios where multiple consumers need to process the same events:

```java
runner.given(
    knative()
        .client(knativeClient)
        .trigger()
        .create("audit-trigger")
        .broker("my-broker")
        .channel("audit-channel")
        .filter("type", "security.audit.event")
        .inNamespace("my-namespace")
);
```

# Channels and subscriptions

Channels are Knative's pub/sub abstraction. They deliver events to multiple subscribers, unlike services that receive events directly from triggers. Citrus supports creating channels and subscriptions as part of the test setup.

Creating a channel is straightforward:

```java
runner.given(
    knative()
        .client(knativeClient)
        .channels()
        .create("audit-channel")
        .inNamespace("my-namespace")
);
```

Subscribing a service to a channel connects the event delivery pipeline:

```java
runner.given(
    knative()
        .client(knativeClient)
        .subscriptions()
        .create("audit-subscription")
        .channel("audit-channel")
        .service("audit-logger")
        .inNamespace("my-namespace")
);
```

With this setup, any event that flows into `audit-channel` — whether from a trigger or directly — is delivered to the `audit-logger` service. You can create multiple subscriptions on the same channel to test fan-out scenarios where the same event is processed by different consumers.

# Putting it all together

When you combine the Citrus Knative capabilities, a typical eventing test follows a clear structure:

1. **Create infrastructure** — a local Knative broker, any required services and triggers, and supporting resources like S3 buckets or Kafka topics.
2. **Trigger the event source** — upload a file, send a message, or call an API that causes the application to produce events.
3. **Validate the events** — receive CloudEvents from the broker and assert on the payload, CloudEvent type, source, subject, and any other attributes.
4. **Clean up** — automatic. Citrus removes brokers, triggers, channels, and subscriptions when the test finishes.

The [Knative eventing sample](/samples/camel-quarkus-knative/) demonstrates this with a Camel Quarkus event source that turns S3 uploads into CloudEvents. The [Knative SSL sample](/samples/camel-quarkus-knative-ssl/) extends the same application with TLS-secured broker communication, showing how SSL configuration is tested without changing the Camel route.

Both samples run entirely on localhost. No Kubernetes cluster, no Knative installation, no AWS account. Yet they exercise the complete event delivery pipeline — from source to broker — with real container-based infrastructure for S3 and faithful local implementations for Knative.

# Knative eventing sample projects

The Citrus sample projects are available in the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub:

- [camel-knative](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel/camel-knative) — local Knative broker with CloudEvent validation
- [camel-knative-ssl](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel/camel-knative-ssl) — TLS-secured broker with mutual TLS

Knative eventing is a powerful model for building event-driven applications, and Citrus makes sure you can test it with the same rigor you apply to any other integration. Give it a try, and let us know what you think!
