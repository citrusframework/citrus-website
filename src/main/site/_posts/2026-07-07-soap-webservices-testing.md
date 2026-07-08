---
layout: post
title: Testing CXF SOAP WebServices with Citrus
short-title: CXF SOAP WebServices Testing
author: Christoph Deppisch
github: christophd
categories: [blog]
---

SOAP is far from dead. Contract-first, WSDL-driven services still power banking systems, healthcare exchanges, government APIs, and countless internal enterprise platforms. If you do integration work in Java, the chances are still quite high that you will encounter SOAP — either as a service you expose, a backend you consume, or a legacy system you bridge to a modern REST API.

![CXF SOAP WebServices testing](/img/assets/soap-ws-testing/featured.png)

Testing these SOAP integrations is where things get interesting. You need to verify SOAP envelopes, validate XML payloads against WSDL schemas, handle SOAP faults, and often coordinate multiple protocols in a single test. Unit tests with mocked stubs only get you so far — real confidence comes from end-to-end tests that exercise the actual SOAP processing stack.

This is exactly where the [Citrus](https://citrusframework.org) integration testing framework excels. Citrus provides a dedicated SOAP WebServices module `citrus-ws` with first-class SOAP support: a WebService client that sends SOAP requests and validates responses, a WebService server that simulates SOAP service backends. The capabilities also include SOAP fault handling on both sides, SOAP attachment and MTOM support, as well as WS-Addressing capabilities. All wrapped in a clean, fluent Java DSL that integrates with JUnit Jupiter.

This post walks through three real-world SOAP testing scenarios, each backed by a working sample project. We start with the basics — testing a SOAP server from the client side — then move to testing a SOAP client by simulating the backend, and finally tackle the most complex case: testing a REST-to-SOAP protocol bridge where Citrus controls both ends of the conversation.

# The Citrus SOAP WebServices module

Before diving into the scenarios, let's look at what the `citrus-ws` module provides. Add it to your project as a test dependency:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-ws</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

The module gives you two main components:

**WebServiceClient** — a SOAP client that constructs SOAP envelopes, sends requests over HTTP, and captures the synchronous response for validation. You provide only the SOAP body content; Citrus handles the envelope wrapping and unwrapping automatically.

**WebServiceServer** — an embedded SOAP server that receives incoming SOAP requests, lets you validate them, and sends back simulated responses (or SOAP faults). It starts an embedded HTTP server with a Spring-WS servlet, ready to accept SOAP messages on a configurable port.

Both components support SOAP 1.1 and 1.2, SOAP attachments, MTOM, WS-Addressing, and custom interceptors. For this post, we focus on the core send/receive/fault capabilities that cover the vast majority of SOAP testing needs.

The fluent DSL for SOAP operations follows the same Given-When-Then pattern as the rest of Citrus:

```java
soap().client(soapClient).send().message().body("<ns:listFruits .../>");
soap().client(soapClient).receive().message().body("<ns:listFruitsResponse .../>");

soap().server(soapServer).receive().message().body("<ns:addFruit .../>");
soap().server(soapServer).send().message().body("<ns:addFruitResponse .../>");

soap().server(soapServer).sendFault().message().faultCode("...").faultString("...");
```

Whether you are acting as a client or a server, the DSL reads naturally and handles the SOAP protocol details behind the scenes.

# Scenario 1: Testing a SOAP WebService as a client

The most straightforward SOAP testing scenario: your application exposes a SOAP WebService, and you want to verify it handles requests correctly and returns the right responses. Citrus acts as the SOAP client.

The sample application is a FruitService implemented with Apache Camel's CXF component in Quarkus. It exposes three WSDL-defined operations — `addFruit`, `deleteFruit`, and `listFruits` — backed by a simple in-memory store. The WSDL contract defines the message types, and WSDL2Java code generation produces the model objects, the service endpoint interface, and fault classes automatically.

The Camel route is elegant in its simplicity:

```java
from("cxf:bean:fruitEndpoint")
    .recipientList(
        simple("bean:fruitService?method=${header.operationName}"));
```

The CXF component receives the SOAP request, extracts the operation name, and the Camel recipient list dispatches to the matching method on the service bean. JAXB marshalling between SOAP XML and Java objects happens transparently.

## Setting up the test

```java
@QuarkusTest
@CitrusSupport
class FruitSoapServiceTest {

    @CitrusEndpoint
    @WebServiceClientConfig(
        requestUrl = "http://localhost:8081/cxf/services/fruits")
    WebServiceClient soapClient;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

Two annotations, two fields, and you are ready to test. `@QuarkusTest` starts the application on the test port. `@CitrusSupport` activates the Citrus framework. The `WebServiceClient` points at the CXF endpoint URL and handles all SOAP envelope construction internally.

## Sending requests and validating responses

Testing the `listFruits` operation is a clean send-receive pair:

```java
@Test
void shouldListFruits() {
    runner.when(
        soap().client(soapClient)
            .send()
            .message()
            .soapAction("listFruits")
            .body("<ns:listFruits xmlns:ns=\"" + TARGET_NS + "\"/>")
    );

    runner.then(
        soap().client(soapClient)
            .receive()
            .message()
            .body("<ns:listFruitsResponse xmlns:ns=\"" + TARGET_NS + "\">" +
                "<fruits>" +
                    "<fruit>" +
                        "<description>Winter fruit</description>" +
                        "<name>Apple</name>" +
                    "</fruit>" +
                    "<fruit>" +
                        "<description>Citrus fruit</description>" +
                        "<name>Orange</name>" +
                    "</fruit>" +
                "</fruits>" +
            "</ns:listFruitsResponse>")
    );
}
```

Notice that you write only the SOAP body XML — Citrus wraps it in the SOAP envelope automatically. The `soapAction("listFruits")` sets the SOAPAction HTTP header, matching the WSDL binding. The `receive()` action validates that the response XML matches the expected structure, including element order and content.

For the `addFruit` operation, the test sends a fruit in the SOAP body and verifies the response contains the updated list:

```java
@Test
void shouldAddFruit() {
    runner.when(
        soap().client(soapClient)
            .send()
            .message()
            .soapAction("addFruit")
            .body("<ns:addFruit xmlns:ns=\"" + TARGET_NS + "\">" +
                "<fruit>" +
                    "<description>Tropical fruit</description>" +
                    "<name>Mango</name>" +
                "</fruit>" +
            "</ns:addFruit>")
    );

    runner.then(
        soap().client(soapClient)
            .receive()
            .message()
            .body("<ns:addFruitResponse xmlns:ns=\"" + TARGET_NS + "\">" +
                "<fruits>...</fruits>" +
            "</ns:addFruitResponse>")
    );
}
```

## Verifying SOAP faults

What happens when the client tries to delete a fruit that does not exist? The service throws a `NoSuchFruitException` (generated from the WSDL fault definition), and CXF translates it into a SOAP fault response. Citrus provides `assertFault()` to handle this case:

```java
@Test
void shouldHandleFruitNotFound() {
    runner.then(
        soap().client(soapClient)
            .assertFault()
            .faultCode("{http://schemas.xmlsoap.org/soap/envelope/}Server")
            .faultString("Fruit \"Pineapple\" does not exist.")
            .when(soap().client(soapClient)
                .send()
                .message()
                .soapAction("deleteFruit")
                .body("<ns:deleteFruit xmlns:ns=\"" + TARGET_NS + "\">" +
                    "<fruit>" +
                        "<description>Tropical fruit</description>" +
                        "<name>Pineapple</name>" +
                    "</fruit>" +
                "</ns:deleteFruit>"))
    );
}
```

The `assertFault()` action wraps a send action and asserts that the response is a SOAP fault rather than a normal response. You specify the expected fault code and fault string, and Citrus validates both. Without `assertFault()`, the SOAP fault would cause an exception in the test — this action intercepts the fault and turns it into a verification point.

The fault code `{http://schemas.xmlsoap.org/soap/envelope/}Server` is the standard SOAP 1.1 server fault code, indicating an error on the server side. You can verify any fault code, including custom codes and fault details defined in your WSDL.

You can find the complete example in the [CXF SOAP WebService sample](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel/camel-cxf-soap).

# Scenario 2: Testing a SOAP client by simulating the backend

The opposite direction is equally common: your application acts as a SOAP client, calling an external WebService. You need to test that your application sends correctly structured SOAP requests and handles responses (and faults) properly. Instead of depending on a real external service, Citrus simulates it with a `WebServiceServer`.

The sample application is a Camel CXF SOAP client in Quarkus. Three Camel routes call an external FruitService SOAP WebService via `direct:` endpoints:

```
direct:listFruits   -->  CXF client  -->  external SOAP listFruits
direct:addFruit     -->  CXF client  -->  external SOAP addFruit
direct:deleteFruit  -->  CXF client  -->  external SOAP deleteFruit
```

The CXF endpoint is configured with a `fruit.service.url` property that defines the URL where the client sends its requests to. The Citrus test overrides this property to point at the Citrus SOAP server instead of a real backend.

```java
public class FruitClientRoutes extends RouteBuilder {

    @ConfigProperty(name = "fruit.service.url")
    String fruitServiceUrl;

    @Produces
    @ApplicationScoped
    @Named
    CxfEndpoint fruitEndpoint() {
        CxfEndpoint fruitEndpoint = new CxfEndpoint();
        fruitEndpoint.setServiceClass(FruitService.class);
        fruitEndpoint.setAddress(fruitServiceUrl);
        return fruitEndpoint;
    }

    @Override
    public void configure() throws Exception {
        from("direct:listFruits")
            .setHeader(CxfConstants.OPERATION_NAME,
                    constant("listFruits"))
            .to("cxf:bean:fruitEndpoint");

        from("direct:addFruit")
            .setHeader(CxfConstants.OPERATION_NAME,
                    constant("addFruit"))
            .to("cxf:bean:fruitEndpoint");

        from("direct:deleteFruit")
            .setHeader(CxfConstants.OPERATION_NAME,
                    constant("deleteFruit"))
            .to("cxf:bean:fruitEndpoint");
    }
}
```

## Setting up server simulation

```java
@QuarkusTest
@CitrusSupport
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class FruitSoapClientTest implements TestActionSupport {

    @CitrusEndpoint
    @WebServiceServerConfig(
        port = 18080, autoStart = true, timeout = 10000)
    WebServiceServer soapServer;

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

The `WebServiceServer` starts an embedded HTTP server on port 18080, ready to receive SOAP messages. The `@TestInstance(PER_CLASS)` annotation shares the server instance across test methods to avoid port conflicts. The `CamelContext` is injected and registered in the Citrus bean registry so the Citrus Camel component can interact with the application's routes.

## Triggering routes and validating SOAP requests

To test the SOAP client, we need to trigger the Camel route and then handle the server side. The Citrus Camel component provides `camel().send()` for this:

```java
@Test
void shouldAddFruit() {
    Fruit mango = new Fruit();
    mango.setName("Mango");
    mango.setDescription("Tropical fruit");

    runner.when(
        camel().send()
            .endpoint("direct:addFruit")
            .fork(true)
            .message(new DefaultMessage(mango))
    );

    runner.then(
        soap().server(soapServer)
            .receive()
            .message()
            .body("<ns:addFruit xmlns:ns=\"" + TARGET_NS + "\">" +
                "<fruit>" +
                    "<description>Tropical fruit</description>" +
                    "<name>Mango</name>" +
                "</fruit>" +
            "</ns:addFruit>")
    );

    runner.then(
        soap().server(soapServer)
            .send()
            .message()
            .body("<ns:addFruitResponse xmlns:ns=\"" + TARGET_NS + "\">" +
                "<fruits>...</fruits>" +
            "</ns:addFruitResponse>")
    );
}
```

The `camel().send()` action sends a `Fruit` POJO to the `direct:addFruit` route, which triggers the CXF client call. The `fork(true)` option is crucial here — the CXF call is synchronous and blocks until the backend responds. Without forking, the test thread would block before it could handle the server-side interactions.

On the server side, Citrus receives the SOAP request and validates the XML body — confirming that the CXF client correctly marshalled the `Fruit` POJO into the expected SOAP XML structure. Then it sends back a simulated response, allowing the CXF call to complete.

## Simulating SOAP faults

Testing error handling from the client's perspective requires simulating a SOAP fault on the server:

```java
@Test
void shouldHandleSoapFault() {
    Fruit pineapple = new Fruit();
    pineapple.setName("Pineapple");
    pineapple.setDescription("Tropical fruit");

    runner.when(
        camel().send()
            .endpoint("direct:deleteFruit", true)
            .fork(true)
            .message(new DefaultMessage(pineapple))
    );

    runner.then(soap().server(soapServer).receive()
            .message().body("..."));

    runner.then(
        soap().server(soapServer)
            .sendFault()
            .message()
            .faultCode(
                "{http://schemas.xmlsoap.org/soap/envelope/}Server")
            .faultString("Fruit \"Pineapple\" does not exist.")
    );

    runner.then(
        camel().receive()
            .endpoint("direct:deleteFruit", true)
            .message()
            .validate((message, context) -> {
                Assertions.assertNotNull(message.getHeader(
                    CamelMessageHeaders.EXCHANGE_EXCEPTION));
                Assertions.assertEquals(
                    "Fruit \"Pineapple\" does not exist.",
                    message.getHeader(
                        CamelMessageHeaders.EXCHANGE_EXCEPTION_MESSAGE));
            })
    );
}
```

The `sendFault()` action on the server sends a SOAP fault response with a fault code and fault string. On the client side, `camel().receive()` captures the Camel exchange result and verifies that the SOAP fault was correctly translated into a Camel exchange exception. The second `true` parameter on `endpoint("direct:deleteFruit", true)` enables exception capture on the endpoint, making the exception details available as message headers.

This pattern verifies the complete error flow: the application sends a SOAP request, the backend returns a fault, and the application handles the fault correctly.

You can find the complete example in the [CXF SOAP Client sample](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel/camel-cxf-soap-client).

# Scenario 3: Testing a REST-to-SOAP protocol bridge

The most complex and rewarding scenario combines both directions. A common enterprise integration pattern is the protocol bridge — an application that exposes a modern REST/JSON API while delegating to a legacy SOAP/XML backend. External consumers see a clean HTTP interface. Behind the scenes, the application translates every request into SOAP XML, calls the backend, and converts the response back to JSON.

Testing this bridge requires controlling both ends simultaneously. Citrus does this by acting as both the HTTP client (sending JSON requests to the REST API) and the SOAP server (simulating the SOAP backend). This dual-role setup lets you verify the entire translation chain in a single test.

The sample application uses Camel's REST DSL as the frontend and the CXF component as the SOAP backend client:

```
GET    /fruits  -->  direct:listFruits  -->  CXF client  -->  SOAP
POST   /fruits  -->  direct:addFruit    -->  CXF client  -->  SOAP
DELETE /fruits  -->  direct:deleteFruit -->  CXF client  -->  SOAP
```

The REST binding mode handles JSON marshalling automatically. The CXF POJO data format handles SOAP marshalling. The application also includes a `SoapFault` exception handler that translates SOAP faults into HTTP 500 responses.

## The dual-role test setup

```java
@QuarkusTest
@CitrusSupport
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class FruitRestSoapBridgeTest implements TestActionSupport {

    @CitrusEndpoint
    @HttpClientConfig(requestUrl = "http://localhost:8081")
    HttpClient fruitRestClient;

    @CitrusEndpoint
    @WebServiceServerConfig(
        port = 18080, autoStart = true, timeout = 10000)
    WebServiceServer soapServer;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

Two Citrus endpoints in one test: an HTTP client for the REST frontend and a SOAP server for the backend. The application sits in between, translating between the two protocols. This test class needs both `citrus-http` and `citrus-ws` dependencies — a natural consequence of testing a multi-protocol bridge.

## Verifying the translation chain

The list fruits test shows the complete flow:

```java
@Test
void shouldListFruits() {
    runner.when(
        http().client(fruitRestClient)
            .send()
            .get("/fruits")
            .fork(true)
    );

    runner.then(
        soap().server(soapServer)
            .receive()
            .message()
            .body("<ns:listFruits xmlns:ns=\"" + TARGET_NS + "\"/>")
    );

    runner.then(
        soap().server(soapServer)
            .send()
            .message()
            .body("<ns:listFruitsResponse xmlns:ns=\"" + TARGET_NS + "\">" +
                "<fruits>" +
                    "<fruit>" +
                        "<description>Winter fruit</description>" +
                        "<name>Apple</name>" +
                    "</fruit>" +
                "</fruits>" +
            "</ns:listFruitsResponse>")
    );
    
    runner.then(
        http().client(fruitRestClient)
            .receive()
            .response(HttpStatus.OK)
            .message()
            .body("""
            [
              { "name": "Apple", "description": "Winter fruit" }
            ]
            """)
    );
}
```

Step by step: the HTTP client sends a GET request to the REST API. The `fork(true)` option runs the HTTP call asynchronously because it blocks until the backend responds — and the test itself controls the backend. The SOAP server receives the translated SOAP request and validates the XML structure, confirming the REST-to-SOAP translation is correct. Then the server sends back a SOAP response, which flows through the CXF unmarshalling and REST JSON serialization back to the HTTP client.

For the add fruit operation, the test verifies the JSON-to-SOAP body translation:

```java
@Test
void shouldAddFruit() {
    runner.when(
        http().client(fruitRestClient)
            .send()
            .post("/fruits")
            .message()
            .body("""
                {"name": "Mango", "description": "Tropical fruit"}
                """)
            .contentType("application/json")
            .fork(true)
    );

    runner.then(
        soap().server(soapServer)
            .receive()
            .message()
            .body("<ns:addFruit xmlns:ns=\"" + TARGET_NS + "\">" +
                "<fruit>" +
                    "<description>Tropical fruit</description>" +
                    "<name>Mango</name>" +
                "</fruit>" +
            "</ns:addFruit>")
    );

    // Server sends back the response...
}
```

The HTTP client sends `{"name": "Mango"}` as JSON. Camel deserializes it to a `Fruit` POJO, and the CXF client marshals that POJO to `<addFruit><fruit><name>Mango</name>...</fruit></addFruit>` SOAP XML. The SOAP server receive step validates this transformation — the critical verification point for any protocol bridge.

## End-to-end error path

The most valuable test in the suite verifies the complete error path — from a SOAP fault on the backend to an HTTP 500 response on the REST side:

```java
@Test
void shouldHandleSoapFault() {
    runner.when(
        http().client(fruitRestClient)
            .send()
            .delete("/fruits")
            .message()
            .body("""
                {"name": "Pineapple", "description": "Tropical fruit"}
                """)
            .contentType("application/json")
            .fork(true)
    );

    runner.then(
        soap().server(soapServer)
            .receive()
            .message()
            .body("<ns:deleteFruit xmlns:ns=\"" + TARGET_NS + "\">...</ns:deleteFruit>")
    );

    runner.then(
        soap().server(soapServer)
            .sendFault()
            .message()
            .faultCode("{http://schemas.xmlsoap.org/soap/envelope/}Server")
            .faultString("Fruit \"Pineapple\" does not exist.")
    );

    runner.then(
        http().client(fruitRestClient)
            .receive()
            .response(500)
            .message()
            .validate((message, context) -> {
                Assertions.assertEquals(
                    "Fruit \"Pineapple\" does not exist.",
                    message.getPayload());
            })
    );
}
```

Four steps, four verification points. The HTTP client sends a DELETE request. The SOAP server validates the translated SOAP request. The SOAP server returns a fault. The HTTP client verifies that the application translated the SOAP fault into an HTTP 500 response with the fault message as the response body.

This single test covers the entire error translation chain: REST request, JSON-to-SOAP translation, SOAP fault, fault-to-HTTP-error translation, and HTTP error response. Without the dual-role Citrus setup, you would need a real SOAP backend that produces faults on demand — making the test fragile, slow, and hard to reproduce.

You can find the complete example in the [REST-to-SOAP Bridge sample](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel/camel-rest-to-soap).

# Type-safe testing with JAXB

All three sample projects used XML payloads as hard coded String values in the test. This can be the source of failures so it is recommended to replace raw XML strings with JAXB marshalling. This approach uses the WSDL2Java generated model objects for type-safe request construction and response validation.

## Building SOAP requests from Java objects

Instead of hand-writing XML, you construct request objects from the generated WSDL2Java types:

```java
Fruit mango = new Fruit();
mango.setName("Mango");
mango.setDescription("Tropical fruit");

AddFruit addFruit = new AddFruit();
addFruit.setFruit(mango);

runner.when(
    soap().client(soapClient)
        .send()
        .message()
        .soapAction("addFruit")
        .body(marshal(new ObjectFactory().createAddFruit(addFruit)))
);
```

The `marshal()` helper serializes the Java object to XML using a JAXB marshaller registered in the Citrus bean registry. No XML strings, no namespace typos, no element ordering mistakes. If the WSDL changes and the generated types are updated, the compiler catches mismatches immediately.

## Validating responses with unmarshalling

On the `receive()` action side, `XmlMarshallingValidationProcessor` deserializes the SOAP response into typed Java objects:

```java
runner.then(
    soap().client(soapClient)
        .receive()
        .message()
        .validate(new XmlMarshallingValidationProcessor
                <JAXBElement<AddFruitResponse>>() {
            @Override
            public void validate(JAXBElement<AddFruitResponse> payload,
                    Map<String, Object> headers, TestContext context) {
                AddFruitResponse response = payload.getValue();
                Assertions.assertTrue(
                    response.getFruits().getFruit().stream()
                        .anyMatch(f -> "Mango".equals(f.getName())));
            }
        })
);
```

You validate the response using Java getters and standard assertions. This catches schema mismatches at compile time rather than at runtime, and the validation logic reads like any other Java test assertion.

## Server-side JAXB validation

The same approach works on the server side. When Citrus acts as a SOAP server, you can validate incoming requests using JAXB unmarshalling:

```java
runner.then(
    soap().server(soapServer)
        .receive()
        .message()
        .validate(new XmlMarshallingValidationProcessor
                <JAXBElement<AddFruit>>() {
            @Override
            public void validate(JAXBElement<AddFruit> payload,
                    Map<String, Object> headers, TestContext context) {
                AddFruit request = payload.getValue();
                Assertions.assertEquals("Mango", request.getFruit().getName());
            }
        })
);
```

And send responses from marshalled objects:

```java
runner.then(
    soap().server(soapServer)
        .send()
        .message()
        .body(marshal(addFruitResponse(
            fruit("Apple", "Winter fruit"),
            fruit("Orange", "Citrus fruit"),
            mango
        )))
);
```

Factory methods like `fruit()` and `addFruitResponse()` keep the test code readable and eliminate XML boilerplate entirely.

Both approaches — raw XML and JAXB — are valid choices. Raw XML is simpler for quick tests and makes the actual SOAP content immediately visible in the test code. JAXB is better for complex schemas where type safety and refactoring support matter. The sample projects demonstrate both so you can pick what fits your situation.

# Key patterns and techniques

Across all three scenarios, several patterns emerge that are worth highlighting.

## Forked sends for synchronous protocols

SOAP over HTTP is synchronous — the client blocks until the server responds. When the test controls the server, you must fork the client send to avoid a deadlock:

```java
http().client(fruitRestClient)
    .send()
    .get("/fruits")
    .fork(true)
```

Without `fork(true)`, the test thread sends the request, blocks waiting for a response, and never reaches the server-side code that would provide that response. This applies whenever the test controls both ends of a synchronous interaction.

## Port isolation for parallel test classes

When multiple test classes start their own SOAP server, each needs a different port. Quarkus test profiles solve this cleanly:

```java
@TestProfile(FruitPojoSoapClientTest.TestConfig.class)
public static class TestConfig implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of("fruit.service.url",
            "http://localhost:18081/services/fruits");
    }
}
```

The test profile overrides the service URL property, pointing the application's CXF client at the test-specific port. Combined with `@TestInstance(PER_CLASS)`, the server instance is shared across methods within a class but isolated between classes.

## The TestActionSupport interface

All test classes that use Citrus DSL builders implement `TestActionSupport`:

```java
public class FruitSoapClientTest implements TestActionSupport {
}
```

This interface provides access to builder methods like `http()`, `soap()`, `camel()`, `async()`, and `createVariables()` without requiring static imports. It is a convenience that keeps the test code clean.

# Beyond the basics

The `citrus-ws` module offers capabilities beyond what these three scenarios cover:

**SOAP attachments** — Citrus can send and validate SOAP attachments (binary or text content referenced in the SOAP body). You specify the attachment content, content type, and content ID, and Citrus attaches it to the SOAP message.

**MTOM support** — for large binary payloads, Citrus supports MTOM (Message Transmission Optimization Mechanism), which uses XOP (XML-binary Optimized Packaging) to efficiently transfer binary data as MIME attachments.

**WS-Addressing** — Citrus can set WS-Addressing headers (To, From, ReplyTo, Action, MessageID) on outgoing SOAP messages and validate them on incoming messages. This is essential for testing asynchronous SOAP patterns and routing scenarios.

**SOAP 1.2** — all client and server components support SOAP 1.2 in addition to SOAP 1.1, selectable through the message factory configuration.

**Custom interceptors** — both client and server support Spring-WS interceptors for cross-cutting concerns like logging, security, or message transformation.

The [Citrus reference documentation](https://citrusframework.org/citrus/reference/html/) covers all of these in detail.

# Getting started

To add SOAP testing capabilities to your Citrus test project, you need the following dependencies:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-ws</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-validation-xml</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

The `citrus-ws` module provides the SOAP client and server components. The `citrus-validation-xml` module enables XML body validation and JAXB marshalling support. For Quarkus integration, add `citrus-quarkus`. For JUnit 5 support, add `citrus-junit-jupiter`.

The three sample projects referenced in this post are available in the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel) repository:

- [camel-cxf-soap](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel/camel-cxf-soap) — Citrus as a SOAP client testing a CXF server
- [camel-cxf-soap-client](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel/camel-cxf-soap-client) — Citrus as a SOAP server simulating an external backend
- [camel-rest-to-soap](https://github.com/citrusframework/citrus-quarkus-examples/tree/main/apache-camel/camel-rest-to-soap) — Citrus in dual-role testing a REST-to-SOAP bridge

Clone the repository, pick a sample, and run `mvn verify` to see the tests in action. Each sample is a self-contained Quarkus application that runs as a standard JUnit 5 test — no external services or infrastructure required.

# Wrapping up

SOAP WebService testing does not have to be painful. The Citrus `citrus-ws` module provides everything you need to test SOAP interactions from both sides of the conversation — as a client verifying server responses, as a server simulating backends, or both simultaneously for protocol bridge testing.

The three scenarios we explored build on each other. Client-side testing validates that your service handles requests and faults correctly. Server simulation verifies that your client sends well-formed SOAP messages and handles errors gracefully. Dual-role bridge testing covers the complete translation chain between protocols. Together, they give you comprehensive confidence in your SOAP integration layer.

The pattern is always the same: configure the Citrus endpoints, write the test actions using the fluent DSL, and let Citrus handle the SOAP envelope processing, XML validation, and fault handling. Whether you prefer raw XML for its transparency or JAXB marshalling for its type safety, the framework supports both approaches equally well.

Give it a try with one of the sample projects, and let us know what you think!
