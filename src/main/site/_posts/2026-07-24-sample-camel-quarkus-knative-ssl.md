---
layout: sample
title: Apache Camel Quarkus - Knative Eventing with SSL/TLS
name: camel-knative-ssl
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing TLS-secured Knative eventing in a Camel Quarkus application using Citrus
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-knative-ssl/
---

Production Knative deployments almost always secure broker communication with TLS. Event sources connect over HTTPS, present client certificates, and validate the broker's identity through a trust chain. This is standard practice for any system that handles sensitive data, but it introduces a testing challenge: how do you verify that your event source correctly negotiates TLS, sends valid CloudEvents over an encrypted channel, and handles the SSL handshake — all without deploying to a Kubernetes cluster?

This post extends the [Knative eventing sample](/samples/camel-quarkus-knative/) with full TLS/SSL support. You will test an Apache Camel Knative event source that delivers CloudEvents to an HTTPS-secured broker, using [Citrus](https://citrusframework.org) as a local SSL-enabled Knative broker and LocalStack for S3 emulation. The Camel route itself does not change at all — the SSL wiring lives entirely in configuration and a CDI bean, and Citrus validates the encrypted communication transparently.

By the end you will have a recipe for testing TLS-secured Knative eventing locally, with mutual TLS between Camel and the broker, zero cloud infrastructure, and full CloudEvent attribute validation.

# The application under test

The application is the same Knative event source from the [plain Knative sample](/samples/camel-quarkus-knative/): it polls an S3 bucket for new files, transforms each file into a CloudEvent, and delivers the event to a Knative broker. The difference is that the broker endpoint is now HTTPS.

```
Amazon S3 upload --> Camel aws-s3-source Kamelet --> CloudEvent transform --> HTTPS Knative broker
```

The Camel route is identical to the non-SSL version:

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

The route does not know or care whether the broker uses HTTP or HTTPS. That separation is the whole point — transport security is a concern of the infrastructure layer, not the integration logic.

## Adding SSL to the Knative component

Two pieces wire SSL into the application. First, a CDI bean provides SSL-aware HTTP client options to the Camel Knative component:

```java
@ApplicationScoped
public class SourceOptions {

    @Named("knativeHttpClientOptions")
    public WebClientOptions knativeHttpClientOptions(
            CamelContext camelContext) {
        return new KnativeSslClientOptions(camelContext);
    }
}
```

`KnativeSslClientOptions` is a Camel-provided class that reads SSL configuration properties from the `CamelContext` and builds a Vert.x `WebClientOptions` instance with the correct keystore, truststore, and hostname verification settings. The `@Named` annotation makes the bean discoverable by the Knative component.

Second, `application.properties` connects the bean and points the broker URL to HTTPS:

```properties
camel.component.knative.environment-path=classpath:knative.json
camel.component.knative.producerFactory.clientOptions=#bean:knativeHttpClientOptions

camel.component.knative.ceOverride[ce-type]=dev.knative.eventing.aws-s3
camel.component.knative.ceOverride[ce-source]=dev.knative.eventing.aws-s3-source
camel.component.knative.ceOverride[ce-subject]=aws-s3-source

%test.k.sink=https://localhost:8443
```

The `producerFactory.clientOptions` setting tells the Knative component to use the SSL-capable HTTP client bean instead of the default plain-HTTP client. The `%test.k.sink` property overrides the broker URL in test mode to use HTTPS on port `8443` — the secure port where the Citrus broker will listen.

The `knative.json` environment file resolves the broker URL from the `k.sink` property:

```json
{
  "resources": [
    {
      "name": "default",
      "type": "event",
      "endpointKind": "sink",
      "url": "{% raw %}{{k.sink:https://localhost:8443}}{% endraw %}",
      "objectApiVersion": "eventing.knative.dev/v1",
      "objectKind": "Broker",
      "objectName": "default"
    }
  ]
}
```

Notice the default URL already uses `https` — this application is designed for TLS from the start, and the test profile simply confirms the port.

# Adding Citrus to the project

The test needs the same Citrus modules as the plain Knative sample, with `citrus-http` replacing `citrus-knative` since the SSL broker is configured as an explicit HTTP server:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-http</artifactId>
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
- **citrus-http** provides the HTTP server builder with SSL support — keystores, truststores, and secure port configuration. This is what lets Citrus act as an HTTPS endpoint that Camel can connect to with mutual TLS.
- **citrus-testcontainers** adds the `@LocalStackContainerSupport` annotation for provisioning a LocalStack container with S3 enabled.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

The key difference from the plain Knative sample is using **citrus-http** directly. The `citrus-knative` module provides a high-level Knative DSL with a local broker abstraction that handles HTTP internally. For the SSL variant, the test configures the HTTP server explicitly to control the TLS settings — keystores, truststores, secure port — which gives full visibility into the SSL handshake setup.

# Understanding the test

The test class structure is familiar from the non-SSL version, with the same annotations and interfaces:

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

The three annotations set up the full test infrastructure: `@QuarkusTest` starts the application, `@CitrusSupport` activates Citrus, and `@LocalStackContainerSupport` provisions a LocalStack container with S3. The `ContainerLifecycleListener` interface lets the test inject properties before the application starts.

## Setting up the HTTPS broker

This is where the SSL version diverges. Instead of using the `knative()` DSL to create a local broker, the test registers an explicit HTTP server with TLS configuration:

```java
@BindToRegistry
public HttpServer knativeBroker = HttpEndpoints.http()
        .server()
        .port(8080)
        .timeout(5000L)
        .securePort(8443)
        .secured(HttpSecureConnection.ssl()
                .keyStore("classpath:keystore/server.jks", "secr3t")
                .trustStore("classpath:keystore/truststore.jks",
                            "secr3t"))
        .autoStart(true)
        .build();
```

This sets up a Citrus HTTP server that listens on both port `8080` (plain HTTP) and port `8443` (HTTPS). The `secured()` call configures TLS with a server keystore and a truststore:

- **`server.jks`** contains the server's private key and certificate. When Camel connects to `https://localhost:8443`, the server presents this certificate during the TLS handshake.
- **`truststore.jks`** contains the test Certificate Authority that both sides trust. The server uses it to verify the client certificate, and the Camel client uses it to verify the server certificate.

The `@BindToRegistry` annotation registers the server in the Citrus component registry so it can be referenced in test actions. The `autoStart(true)` setting ensures the server is running before the test method executes — Camel needs the HTTPS endpoint available as soon as the route starts.

All certificates are self-signed and ship with the test under `src/test/resources/keystore/`. They exist solely for testing — no real CA or production certificates are involved.

## Injecting SSL and S3 properties at runtime

When the LocalStack container starts, the lifecycle listener does double duty: it creates the S3 bucket and injects both the S3 connection properties and the SSL client configuration:

```java
@Override
public Map<String, String> started(LocalStackContainer container) {
    S3Client s3Client = container.getClient(AwsService.S3);
    s3Client.createBucket(
            builder -> builder.bucket(s3BucketName));

    return Map.ofEntries(
        // S3 connection
        Map.entry("camel.kamelet.aws-s3-source.accessKey",
                  container.getAccessKey()),
        Map.entry("camel.kamelet.aws-s3-source.secretKey",
                  container.getSecretKey()),
        Map.entry("camel.kamelet.aws-s3-source.region",
                  container.getRegion()),
        Map.entry("camel.kamelet.aws-s3-source.bucketNameOrArn",
                  s3BucketName),
        Map.entry("camel.kamelet.aws-s3-source.uriEndpointOverride",
                  container.getServiceEndpoint().toString()),
        Map.entry("camel.kamelet.aws-s3-source.overrideEndpoint",
                  "true"),
        Map.entry("camel.kamelet.aws-s3-source.forcePathStyle",
                  "true"),
        // SSL handshake
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
}
```

The first block of properties is identical to the non-SSL sample — it points the `aws-s3-source` Kamelet at the LocalStack container. The second block configures the SSL client that `KnativeSslClientOptions` reads from the `CamelContext`:

- **`ssl.enabled`** activates TLS on the Knative HTTP client.
- **`ssl.verify.hostname`** is set to `false` because the test certificate's Common Name does not match `localhost`. In production you would leave this enabled.
- **`ssl.key.path`** and **`ssl.key.cert.path`** provide the client's private key and certificate for mutual TLS. The server validates these against its truststore.
- **`ssl.truststore.path`** and **`ssl.truststore.password`** tell the client which CA to trust when verifying the server's certificate.

This property injection pattern is what makes the test self-contained. The test controls both sides of the TLS connection — it provides the server certificate through the `HttpServer` configuration and the client certificate through these properties — so the handshake succeeds without any external PKI infrastructure.

## Validating CloudEvents over HTTPS

With the HTTPS broker running and the SSL properties injected, the test follows the same given-when-then flow as the non-SSL version. The test uploads a file to S3, then validates the CloudEvent that Camel delivers to the secure broker:

```java
@Test
void shouldProduceCloudEventsOverSsl() {
    runner.when(this::uploadS3File);

    runner.then(
        http().server(knativeBroker)
                .receive()
                .post()
                .message()
                .body(s3Data)
                .header("ce-id",
                    "@matches([0-9A-Z]{15}-[0-9]{16})@")
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
}
```

The test uses the `http().server()` DSL instead of the `knative()` DSL, which gives explicit control over the HTTP exchange. Three actions form the validation:

1. **Upload** — the `uploadS3File` method uses the AWS SDK to upload `message.txt` with content `Hello Knative!` to the LocalStack S3 bucket. The Camel Kamelet detects the new object and triggers the route.

2. **Receive and validate** — the `receive().post()` action waits for Camel to deliver the CloudEvent to the HTTPS broker. It validates the message body matches the uploaded content and checks four CloudEvent headers:
   - **`ce-id`** matches a generated identifier pattern using Citrus's `@matches()@` validation expression.
   - **`ce-type`**, **`ce-source`**, and **`ce-subject`** match the values configured in `application.properties`.

3. **Respond** — the `send().response(HttpStatus.OK)` action completes the HTTP exchange. Without this, Camel would see a timeout and potentially retry the delivery.

The fact that this exchange happens over TLS is transparent to the test assertions. The `knativeBroker` server handles the SSL handshake automatically based on its configuration. If the handshake fails — wrong certificate, expired CA, mismatched hostname — the test fails before the receive action even executes, giving you immediate feedback about SSL configuration problems.

# How it all fits together

When you run the test with `./mvnw verify`, the sequence of events unfolds:

1. **`@LocalStackContainerSupport` launches** a LocalStack container with the S3 service. The lifecycle listener creates the `knative-bucket` and returns both the S3 connection properties and the SSL client properties.
2. **Quarkus starts** the application in test mode. The `KnativeSslClientOptions` bean reads the injected SSL properties and configures a TLS-enabled HTTP client.
3. **The Citrus HTTP server** (`knativeBroker`) starts on port `8443` with the server keystore, ready to accept HTTPS connections.
4. **The Camel route** begins polling the S3 bucket via the `aws-s3-source` Kamelet.
5. **The test uploads** `message.txt` with content `Hello Knative!` to the S3 bucket via the AWS SDK.
6. **The Kamelet detects** the new object and consumes it from the bucket.
7. **The route transforms** the content into a CloudEvent with the configured `ce-type`, `ce-source`, and `ce-subject` attributes.
8. **Camel negotiates a TLS handshake** with the Citrus HTTPS broker, presenting the client certificate and verifying the server certificate against the shared truststore.
9. **Camel posts** the CloudEvent to `https://localhost:8443` over the encrypted channel.
10. **Citrus receives** the CloudEvent, validates the payload and all CloudEvent headers, and responds with `200 OK`.

The entire test runs on localhost with self-signed certificates. No Kubernetes cluster, no Knative installation, no real AWS account, no certificate authority. Yet it exercises the complete TLS-secured event delivery flow — from S3 upload through SSL handshake to CloudEvent validation.

# Why testing SSL matters

It is tempting to skip TLS in tests and assume that if the plain HTTP flow works, the SSL version will too. In practice, SSL configuration is one of the most common sources of deployment failures. A missing truststore entry, an expired certificate, a hostname mismatch, or a wrong keystore password can all cause the application to fail silently or with cryptic error messages that are hard to diagnose in a deployed environment.

By testing with real TLS, you verify:

- The `KnativeSslClientOptions` bean correctly reads and applies the SSL properties.
- The client certificate and private key are in the right format and location.
- The truststore contains the correct CA certificate.
- The entire certificate chain is valid and the handshake completes successfully.
- The CloudEvent payload and headers survive the encrypted transport unchanged.

This test catches SSL configuration errors during development, not during deployment. If you rotate certificates, add a new CA, or change keystore formats, the test tells you immediately whether the configuration is correct.

# Where to go from here

This example covers TLS with self-signed certificates, but the same pattern applies to more advanced SSL scenarios:

- **Certificate rotation testing**: Swap the test keystores between test runs to verify that the application handles certificate updates without downtime.
- **Mutual TLS enforcement**: Configure the Citrus server to require client certificates and verify that Camel correctly presents them. Remove the client certificate and confirm the handshake fails.
- **Mixed transport**: Test a setup where some brokers use HTTPS and others use plain HTTP, verifying that the Knative component applies SSL settings selectively.
- **Plain HTTP baseline**: Compare with the [non-SSL Knative sample](/samples/camel-quarkus-knative/) to see how the same route works without TLS. The route code is identical — only the configuration differs.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers HTTP server configuration, SSL support, Testcontainers integration, and the many other transports that Citrus supports.
