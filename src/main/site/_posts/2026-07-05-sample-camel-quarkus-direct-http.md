---
layout: sample
title: Apache Camel Quarkus - HTTP Service Virtualization
name: camel-direct-http
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel routes with external HTTP dependencies using Citrus service virtualization
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-direct-http/
---

Most integration routes do not live in isolation. They call external HTTP services — a translation API, a payment gateway, a notification service. Testing these routes means you need a way to simulate those external dependencies: validate the requests your route makes, control the responses it receives, and verify the end-to-end flow without depending on real services.

This is where service virtualization comes in. Instead of calling the real service, your test starts a simulated HTTP server that behaves exactly the way you tell it to. You get full control over request validation, response content, error scenarios, and timing — all within a deterministic, repeatable test.

This post walks you through testing an Apache Camel route with an external HTTP dependency in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus spins up a simulated HTTP server, validates the requests Camel sends to it, sends back controlled responses, and verifies the final output. 

By the end you will have a recipe for testing any Camel route that calls external HTTP services.

# The application under test

The example application implements a translation pipeline. It receives a text message, optionally calls an external translation service based on a language header, transforms the result to uppercase with a prefix, and sends it to an output endpoint.

```
Direct Endpoint (direct:words-in) --> Translation Route --> HTTP Service --> Transform --> Mock Endpoint (mock:words-out)
```

The logic is split into two interconnected Camel routes. The main route handles the overall pipeline:

```java
from("direct:words-in")
    .to("direct:translate")
    .setBody(exchange -> ">> " + exchange.getIn().getBody().toString().toUpperCase())
    .to("mock:words-out");
```

The translation route handles the HTTP service call with content-based routing:

```java
from("direct:translate")
    .choice()
        .when(simple("${header.lang} != null"))
            .setVariable("lang", simple("${header.lang}"))
            .removeHeaders("*")
            .setHeader(Exchange.HTTP_METHOD, constant("POST"))
            .setHeader(Exchange.HTTP_QUERY, simple("lang=${variable.lang}"))
            .to("http://{{camel.translate.service.host}}:{{camel.translate.service.port}}/translate")
            .convertBodyTo(String.class)
        .otherwise()
            .setBody(simple("${body}"));
```

There is a lot going on in the translation route, so let's walk through it.

The **Content Based Router** (`choice().when()`) checks whether the incoming message carries a `lang` header. If it does, the route calls the translation service. If not, it passes the original message through unchanged.

Before making the HTTP call, the route stores the `lang` value in a Camel **variable** using `setVariable()`. This is important because the next step — `removeHeaders("*")` — clears all message headers. Without saving the language first, it would be lost. Variables survive header removal, so the route can reconstruct the query parameter later with `simple("lang=${variable.lang}")`.

The HTTP endpoint URL uses **property placeholders**, resolved from `application.properties`:

```properties
camel.translate.service.host=localhost
camel.translate.service.port=9001
```

This design — route composition, content-based routing, variable management, and property-driven configuration — is representative of real-world Camel applications. Now let's test it.

# Adding Citrus to the project

This example needs the Citrus HTTP module for service virtualization and the Camel module for direct endpoint access:

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
    <artifactId>citrus-http</artifactId>
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
- **citrus-camel** provides the `camel:` URI scheme for sending messages into Camel routes and CamelContext sharing.
- **citrus-http** provides the HTTP server endpoint that simulates the external translation service.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

You also need `camel-quarkus-junit` for mock endpoint access, as in the [direct endpoint sample](/samples/camel-quarkus-direct/). No brokers, no containers — just an in-process HTTP server managed by Citrus.

# Setting up the test class

The test class wires together the shared CamelContext and the simulated HTTP server:

```java
@QuarkusTest
@CitrusSupport
class QuarkusApplicationTest extends CamelQuarkusTestSupport
        implements TestActionSupport {

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    @CitrusEndpoint
    @HttpServerConfig(autoStart = true, port = 9001)
    HttpServer translateServer;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

The CamelContext is shared with `@BindToRegistry` as in the other Camel samples. The new element here is the `HttpServer`.

The `@HttpServerConfig` annotation configures a Citrus HTTP server on port 9001 with `autoStart = true`, which means the server is ready before any test method runs. The port must match the `camel.translate.service.port` property in `application.properties` — this is how the Camel route's HTTP call reaches the Citrus server instead of a real translation service.

When the Camel route executes `to("http://localhost:9001/translate")`, the request lands on this simulated server. The test then has full control: it can inspect the request, validate its contents, and decide what response to send back.

# Writing the test

The test orchestrates a five-step flow: set up the mock expectation, send a message into the route, validate the HTTP request Camel makes, simulate the HTTP response, and verify the final output.

```java
@Test
void shouldHandleEvents() {
    MockEndpoint mockEndpoint = getMockEndpoint("mock:words-out");
    mockEndpoint.expectedBodiesReceived(">> HOWDY");

    runner.when(
        send()
            .fork(true)
            .endpoint("camel:direct:words-in")
            .message()
            .header("lang", "us-texas")
            .body("Hello")
    );

    runner.when(
        http().server(translateServer)
            .receive()
            .post("/translate")
            .message()
            .queryParam("lang", "us-texas")
            .body("Hello")
    );

    runner.then(
        http().server(translateServer)
            .send()
            .response(200)
            .message()
            .body("Howdy")
    );

    runner.then(
        context -> {
            try {
                mockEndpoint.assertIsSatisfied();
            } catch (InterruptedException e) {
                throw new CitrusRuntimeException(
                    "Failed to verify mock endpoint", e);
            }
        }
    );
}
```

Let's walk through each step.

## Triggering the route

```java
runner.when(
    send()
        .fork(true)
        .endpoint("camel:direct:words-in")
        .message()
        .header("lang", "us-texas")
        .body("Hello")
);
```

The test sends the string `"Hello"` to the `camel:direct:words-in` endpoint with a `lang` header set to `"us-texas"`. The `fork(true)` option sends the message asynchronously because the `direct:` component is synchronous and the route will block while waiting for the HTTP response from the translation service.

At this point, the Camel route begins executing. The main route delegates to the translation sub-route, which detects the `lang` header, prepares the HTTP request, and calls `http://localhost:9001/translate?lang=us-texas` with the body `"Hello"`. That request arrives at the Citrus HTTP server.

## Validating the HTTP request

```java
runner.when(
    http().server(translateServer)
        .receive()
        .post("/translate")
        .message()
        .queryParam("lang", "us-texas")
        .body("Hello")
);
```

The Citrus HTTP server receives the request from the Camel route and validates it. This step verifies four things at once:

- The HTTP method is POST
- The path is `/translate`
- The query parameter `lang` equals `"us-texas"`
- The request body is `"Hello"`

If any of these checks fail, Citrus reports exactly what was expected versus what was received. This is a powerful validation point — it confirms that the Camel route correctly prepared the HTTP request, including the variable-to-query-parameter conversion and the header management logic.

## Simulating the response

```java
runner.then(
    http().server(translateServer)
        .send()
        .response(200)
        .message()
        .body("Howdy")
);
```

After validating the request, the Citrus server sends back a 200 OK response with the body `"Howdy"`. This simulates the translation service returning the translated text. The Camel route receives this response, converts it to a string, and passes it back to the main route.

The main route then transforms `"Howdy"` to `">> HOWDY"` (uppercase with prefix) and sends it to `mock:words-out`.

## Verifying the final output

```java
runner.then(
    context -> {
        try {
            mockEndpoint.assertIsSatisfied();
        } catch (InterruptedException e) {
            throw new CitrusRuntimeException(
                "Failed to verify mock endpoint", e);
        }
    }
);
```

Finally, the test verifies that the mock endpoint received `">> HOWDY"` — confirming that the entire pipeline worked correctly: the route sent the right request to the translation service, processed the response, applied the transformation, and delivered the final result.

# The power of service virtualization

The simulated HTTP server is what makes this test self-contained. Without it, you would need the real translation service running during tests — which introduces network dependencies, authentication, rate limits, and flaky failures.

With Citrus, you get full control over both sides of the HTTP interaction:

**Request validation** — verify the method, path, query parameters, headers, and body of every request your route makes. This catches bugs in request preparation that would otherwise only surface in production.

**Response simulation** — return any status code, body, or header you want. Test the happy path with 200 OK, error handling with 500, not-found behavior with 404, or timeout handling by adding a delay.

**Deterministic behavior** — the simulated server always responds the same way, eliminating flaky tests caused by network issues or external service changes.

This pattern applies to any external dependency your Camel route calls — REST APIs, SOAP services, webhooks, or any HTTP-based interaction.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events:

1. **Citrus starts** the HTTP server on port 9001.
2. **Quarkus starts** the application in test mode with the translation service configured to `localhost:9001`.
3. **Apache Camel discovers** both routes and registers the direct and mock endpoints.
4. **The test sends** `"Hello"` with `lang=us-texas` to `camel:direct:words-in`.
5. **The main route** delegates to the translation sub-route.
6. **The translation route** detects the `lang` header, prepares the HTTP request, and calls the Citrus server.
7. **Citrus validates** the request (POST, /translate, query param, body).
8. **Citrus responds** with 200 OK and body `"Howdy"`.
9. **The main route** transforms the response to `">> HOWDY"` and sends it to the mock endpoint.
10. **The test verifies** the mock endpoint received the expected message.

The entire test runs in under three seconds with no external services, no containers, and no network dependencies.

# Where to go from here

This example tests the happy path with a successful translation. Here are several directions you can take it:

- **Error scenarios**: Simulate a 500 error from the translation service and verify that the Camel route handles it correctly — does it fall back to the original text, retry, or route to a dead-letter endpoint?
- **Missing header path**: Send a message without the `lang` header and verify that the route skips the HTTP call entirely and returns the original message transformed to uppercase.
- **Timeout testing**: Add a delay to the simulated response and verify that the Camel route's timeout handling works as expected.
- **JSON payloads**: If your translation service uses JSON request and response bodies, add the `citrus-validation-json` module for structured payload validation.
- **Multiple services**: If your route calls several external services, set up multiple Citrus HTTP servers on different ports and test the complete orchestration.
- **Simpler routes**: If your Camel route does not call external services, the [direct endpoint sample](/samples/camel-quarkus-direct/) demonstrates testing with just direct and mock endpoints — no HTTP server needed.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers HTTP endpoints, Camel integration, service virtualization, and the many other transports that Citrus supports.
