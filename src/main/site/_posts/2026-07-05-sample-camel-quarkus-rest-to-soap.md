---
layout: sample
title: Apache Camel Quarkus - REST to SOAP Bridge
name: camel-rest-to-soap
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing a Camel REST-to-SOAP protocol bridge in Quarkus with Citrus HTTP client and SOAP server
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-rest-to-soap/
---

A common enterprise integration pattern is the protocol bridge — a component that exposes a modern REST/JSON API while delegating to a legacy SOAP/XML backend. External consumers see a clean HTTP interface with JSON payloads. Behind the scenes, Apache Camel translates every request into SOAP XML, calls the backend service, and converts the XML response back to JSON. The consumers never know they are talking to a SOAP service.

Testing this kind of bridge is uniquely challenging. You need to verify the entire translation chain: JSON in, SOAP XML out, SOAP XML back, JSON out. You need to test happy paths across multiple HTTP verbs. And you need to verify that SOAP faults from the backend are correctly translated into HTTP error responses for the REST consumer.

This post walks you through testing an Apache Camel REST-to-SOAP bridge in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. Citrus plays two roles simultaneously in this test: an HTTP client that sends JSON requests to the REST API, and a SOAP server that simulates the backend WebService. This dual-role setup lets you verify the complete end-to-end flow — from the REST request through the protocol translation to the SOAP backend and back.

By the end you will have a recipe for testing any protocol bridge where Camel mediates between different message formats and transports.

# The application under test

The example application exposes a FruitService REST API that delegates to a SOAP backend. Three HTTP operations map to three SOAP operations on the same FruitService WSDL contract used in the [CXF SOAP server](/samples/camel-quarkus-cxf-soap/) and [CXF SOAP client](/samples/camel-quarkus-cxf-soap-client/) samples:

```
GET    /fruits  -->  direct:listFruits  -->  CXF client  -->  SOAP listFruits
POST   /fruits  -->  direct:addFruit    -->  CXF client  -->  SOAP addFruit
DELETE /fruits  -->  direct:deleteFruit -->  CXF client  -->  SOAP deleteFruit
```

The entire bridge fits in a single route builder:

```java
public class FruitClientRoutes extends RouteBuilder {

    @ConfigProperty(name = "fruit.service.url")
    String fruitServiceUrl;

    @Produces
    @ApplicationScoped
    @Named
    CxfEndpoint fruitSoapClient() {
        CxfEndpoint fruitEndpoint = new CxfEndpoint();
        fruitEndpoint.setServiceClass(FruitService.class);
        fruitEndpoint.setAddress(fruitServiceUrl);
        return fruitEndpoint;
    }

    @Override
    public void configure() throws Exception {
        onException(SoapFault.class)
                .handled(true)
                .process(this::processSoapFault);

        restConfiguration()
                .bindingMode(RestBindingMode.json);

        rest("/fruits").post()
                .type(Fruit.class)
                .produces(APPLICATION_JSON_CONTENT_TYPE)
                .to("direct:addFruit");
        rest("/fruits").get()
                .type(Fruit.class)
                .produces(APPLICATION_JSON_CONTENT_TYPE)
                .to("direct:listFruits");
        rest("/fruits").delete()
                .type(Fruit.class)
                .produces(APPLICATION_JSON_CONTENT_TYPE)
                .to("direct:deleteFruit");

        from("direct:listFruits")
                .setHeader(CxfConstants.OPERATION_NAME,
                        constant("listFruits"))
                .to("cxf:bean:fruitSoapClient");

        from("direct:addFruit")
                .setHeader(CxfConstants.OPERATION_NAME,
                        constant("addFruit"))
                .to("cxf:bean:fruitSoapClient");

        from("direct:deleteFruit")
                .setHeader(CxfConstants.OPERATION_NAME,
                        constant("deleteFruit"))
                .to("cxf:bean:fruitSoapClient");
    }

    private void processSoapFault(Exchange exchange) {
        SoapFault caused = exchange.getProperty(
                Exchange.EXCEPTION_CAUGHT, SoapFault.class);
        exchange.getIn().setBody(caused.getMessage());
        exchange.getIn().setHeader(Exchange.HTTP_RESPONSE_CODE, 500);
    }
}
```

There are several layers of translation happening here, so let's walk through them.

## The REST facade

The `restConfiguration().bindingMode(RestBindingMode.json)` setting tells Camel to automatically marshal and unmarshal JSON request and response bodies using Jackson. When a POST request arrives with `{"name": "Mango", "description": "Tropical fruit"}`, Camel deserializes it into a `Fruit` POJO before the route logic runs. When the route produces a response, Camel serializes the Java objects back to JSON before sending the HTTP response.

The `rest("/fruits")` block defines three HTTP operations — GET, POST, and DELETE — all on the `/fruits` path. Each operation routes to a `direct:` endpoint that bridges to the SOAP backend. The `.type(Fruit.class)` declaration tells the REST binding which Java type to deserialize the request body into.

## The SOAP backend call

Each `direct:` route sets the `CxfConstants.OPERATION_NAME` header and forwards to the CXF endpoint. The CXF component in POJO data format mode takes the Java `Fruit` object, marshals it to SOAP XML matching the WSDL schema, sends the request, and unmarshals the SOAP XML response back to Java objects. The REST binding mode then converts those Java objects to JSON for the HTTP response.

The full translation chain for a POST request looks like this:

```
JSON {"name":"Mango"} --> Fruit POJO --> SOAP <addFruit><fruit>...</fruit></addFruit>
                                                    |
SOAP <addFruitResponse><fruits>...</fruits></addFruitResponse> --> Fruits POJO --> JSON [...]
```

## SOAP fault translation

The `onException(SoapFault.class)` handler catches SOAP faults from the backend and translates them into HTTP 500 responses. The `.handled(true)` directive prevents the exception from propagating further. The `processSoapFault` method extracts the fault message string and sets it as the HTTP response body with a 500 status code.

Without this handler, a SOAP fault would bubble up as an unhandled exception and produce a generic Quarkus error page. With it, REST consumers get a clear error message that originated from the SOAP backend.

# Adding Citrus to the project

This example needs Citrus modules for both the HTTP client side and the SOAP server side:

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
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-validation-json</artifactId>
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
- **citrus-http** provides the HTTP client that sends JSON requests to the REST API and validates JSON responses.
- **citrus-ws** provides the SOAP server that simulates the backend WebService, including SOAP fault simulation.
- **citrus-validation-xml** enables structural XML validation and JAXB unmarshalling for SOAP request verification.
- **citrus-validation-json** handles JSON response body validation on the HTTP side.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

This is the first sample on this site that uses both **citrus-http** and **citrus-ws** in the same test. The HTTP module drives the REST client side while the WS module handles the SOAP server side — matching the dual-protocol nature of the bridge under test.

# Setting up the test class

The test class configures both a Citrus HTTP client and a Citrus SOAP server:

```java
@QuarkusTest
@CitrusSupport
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class FruitRestSoapBridgeTest implements TestActionSupport {

    @CitrusEndpoint
    @HttpClientConfig(requestUrl = "http://localhost:8081")
    HttpClient fruitRestClient;

    @CitrusEndpoint
    @WebServiceServerConfig(port = 18080, autoStart = true,
            timeout = 10000)
    WebServiceServer soapServer;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

The `HttpClient` sends requests to the Quarkus REST API on port 8081 (the Quarkus test port). The `WebServiceServer` starts on port 18080 — the same port that `fruit.service.url` points to in `application.properties`. When the Camel CXF client sends a SOAP request to the backend, it reaches this simulated server.

The `@TestInstance(TestInstance.Lifecycle.PER_CLASS)` annotation shares the SOAP server instance across test methods, avoiding port conflicts that would occur if JUnit created a new server for each test.

With this setup, the test controls both ends of the bridge: it initiates requests on the REST side and handles them on the SOAP side.

# Testing the happy path

Each test follows a four-step pattern: send an HTTP request, validate the SOAP request on the backend, send a simulated SOAP response, and (implicitly) let the REST response flow back to the HTTP client.

## Listing fruits

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
                    "<fruit>" +
                        "<description>Citrus fruit</description>" +
                        "<name>Orange</name>" +
                    "</fruit>" +
                "</fruits>" +
            "</ns:listFruitsResponse>")
    );
}
```

The `http().client(fruitRestClient).send().get("/fruits")` action sends an HTTP GET request to the Camel REST API. The `fork(true)` option is essential here — the HTTP call blocks until the backend responds, and the backend is the Citrus SOAP server that the test itself controls. Without forking, the test thread would be stuck waiting for a response it has not sent yet.

The `soap().server(soapServer).receive()` action waits for the SOAP request that the Camel CXF client sends to the backend. Citrus validates the body matches the expected `listFruits` XML structure. This is the first verification point: it confirms that the Camel route correctly translated the HTTP GET into a SOAP `listFruits` operation.

The `soap().server(soapServer).send()` action sends back a simulated SOAP response containing Apple and Orange. CXF unmarshals this response to Java objects, and the REST binding mode serializes them to JSON for the HTTP response. The HTTP client receives the JSON response, completing the round trip.

## Adding a fruit

```java
@Test
void shouldAddFruit() {
    runner.when(
        http().client(fruitRestClient)
            .send()
            .post("/fruits")
            .message()
            .body("""
                {
                  "name": "Mango",
                  "description": "Tropical fruit"
                }
                """)
            .contentType(APPLICATION_JSON_CONTENT_TYPE)
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

This test verifies the full JSON-to-SOAP translation. The HTTP client sends `{"name": "Mango", "description": "Tropical fruit"}` as a JSON POST body. The Camel REST binding mode deserializes it into a `Fruit` POJO. The CXF client marshals that POJO into `<addFruit><fruit><name>Mango</name>...</fruit></addFruit>` SOAP XML.

The SOAP server receive step validates the XML structure — confirming that the JSON fields were correctly mapped to the SOAP schema elements. This is the critical verification point for a protocol bridge: the translation from one format to the other must be exact.

# Testing SOAP fault translation

The most valuable test verifies the error path end to end — from a SOAP fault on the backend to an HTTP 500 response on the REST side:

```java
@Test
void shouldHandleSoapFault() {
    runner.when(
        http().client(fruitRestClient)
            .send()
            .delete("/fruits")
            .message()
            .body("""
                {
                  "name": "Pineapple",
                  "description": "Tropical fruit"
                }
                """)
            .contentType(APPLICATION_JSON_CONTENT_TYPE)
            .fork(true)
    );

    runner.then(
        soap().server(soapServer)
            .receive()
            .message()
            .body("<ns:deleteFruit xmlns:ns=\"" + TARGET_NS + "\">" +
                "<fruit>" +
                    "<description>Tropical fruit</description>" +
                    "<name>Pineapple</name>" +
                "</fruit>" +
            "</ns:deleteFruit>")
    );

    runner.then(
        soap().server(soapServer)
            .sendFault()
            .message()
            .faultCode(
                "{http://schemas.xmlsoap.org/soap/envelope/}Server")
            .faultString(
                "Fruit \"Pineapple\" does not exist.")
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

This test has four steps that trace the complete error flow.

**Step 1** — the HTTP client sends a DELETE request with a Pineapple fruit that does not exist on the backend.

**Step 2** — the SOAP server validates the incoming `deleteFruit` SOAP request to confirm the CXF client sent the correct XML.

**Step 3** — the SOAP server sends back a SOAP fault with a `Server` fault code and the message `"Fruit \"Pineapple\" does not exist."`. The CXF client on the Camel side receives this fault and throws a `SoapFault` exception.

**Step 4** — the Camel route's `onException(SoapFault.class)` handler catches the exception, extracts the fault message, and returns it as an HTTP 500 response. The Citrus HTTP client validates the 500 status code and the error message in the response body.

This single test verifies the entire error translation chain: REST request → SOAP request → SOAP fault → HTTP 500 with fault message. Without the dual-role Citrus setup, you would need a real SOAP backend that can produce faults on demand — which makes the test fragile and hard to reproduce.

# JAXB-based testing

The example also includes a POJO test class that uses JAXB marshalling on the SOAP server side and `JsonSupport.marshal()` for JSON request generation. This approach avoids XML and JSON string literals entirely.

## Sending JSON from Java objects

```java
Fruit mango = new Fruit();
mango.setName("Mango");
mango.setDescription("Tropical fruit");

runner.when(
    http().client(fruitRestClient)
        .send()
        .post("/fruits")
        .message()
        .body(JsonSupport.marshal(mango))
        .contentType(APPLICATION_JSON_CONTENT_TYPE)
        .fork(true)
);
```

The `JsonSupport.marshal()` helper serializes the `Fruit` object to JSON using the registered Jackson `ObjectMapper`. This eliminates the risk of typos in hand-written JSON strings and keeps the request body in sync with the model class.

## Validating SOAP requests as Java objects

```java
runner.then(
    soap().server(soapServer)
        .receive()
        .message()
        .validate(
            new XmlMarshallingValidationProcessor
                    <JAXBElement<AddFruit>>() {
                @Override
                public void validate(
                        JAXBElement<AddFruit> payload,
                        Map<String, Object> headers,
                        TestContext context) {
                    AddFruit request = payload.getValue();
                    Assertions.assertEquals("Mango",
                            request.getFruit().getName());
                    Assertions.assertEquals("Tropical fruit",
                            request.getFruit().getDescription());
                }
            })
);
```

The `XmlMarshallingValidationProcessor` unmarshals the incoming SOAP request into a typed `JAXBElement<AddFruit>`. You validate the request using Java getters and assertions instead of XML string comparison. If the WSDL changes and the generated types are updated, the compiler catches any mismatches.

## Sending SOAP responses as Java objects

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

The `marshal()` helper with `addFruitResponse()` and `fruit()` factory methods from the `FruitPojoHelper` class constructs the SOAP response from Java objects. No XML strings anywhere — the JAXB marshaller handles the serialization.

## Port isolation with test profiles

Since both the XML and POJO test classes start a SOAP server, they need different ports to avoid conflicts:

```java
@TestProfile(FruitPojoRestSoapBridgeTest.TestConfig.class)

public static class TestConfig implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of("fruit.service.url",
                "http://localhost:18081/services/fruits");
    }
}
```

The `@TestProfile` overrides the CXF client URL to point at port 18081 instead of 18080. Each test class runs its own SOAP server on its own port.

# How it all fits together

When you run the tests with `./mvnw clean test`, here is the sequence of events:

1. **Citrus starts** the SOAP server on port 18080.
2. **Quarkus starts** the application with the REST facade and CXF client, with the backend URL pointing to the Citrus server.
3. **The HTTP client** sends a JSON request to the Camel REST API.
4. **Camel REST binding** deserializes the JSON to a Java object.
5. **The CXF client** marshals the object to SOAP XML and sends it to the Citrus server.
6. **Citrus validates** the SOAP request (XML comparison or JAXB unmarshalling).
7. **Citrus sends** a simulated SOAP response (or a SOAP fault).
8. **The CXF client** unmarshals the response to Java objects.
9. **Camel REST binding** serializes the objects to JSON for the HTTP response.
10. **The HTTP client** receives and validates the JSON response (or the HTTP 500 error).

The entire test suite runs in under five seconds with no external services. Seven tests across two test classes verify all three operations plus SOAP fault handling.

# The bridge testing pattern

This dual-role pattern — Citrus as both HTTP client and SOAP server — applies to any protocol bridge, not just REST-to-SOAP. If your Camel route bridges:

- **REST to JMS** — use a Citrus HTTP client and a Citrus JMS endpoint
- **REST to Kafka** — use a Citrus HTTP client and a Citrus Kafka endpoint
- **JMS to SOAP** — use a Citrus JMS endpoint and a Citrus SOAP server
- **Kafka to REST** — use a Citrus Kafka endpoint and a Citrus HTTP server

The idea is always the same: Citrus controls both ends of the bridge so you can verify the translation in isolation without depending on real external systems.

# Where to go from here

This example covers the core bridge testing pattern. Here are several directions you can take it:

- **Response validation**: Add `http().client(fruitRestClient).receive()` steps to validate the JSON response body after each SOAP response — verify that the JSON structure matches what your REST consumers expect.
- **Content-Type negotiation**: Test what happens when the REST client sends XML instead of JSON, or requests XML responses via the `Accept` header.
- **Error code mapping**: Extend the `processSoapFault` method to map different SOAP fault codes to different HTTP status codes (400 for client faults, 503 for service unavailable) and test each mapping.
- **SOAP-only testing**: If you need to test the SOAP backend in isolation (without the REST facade), the [CXF SOAP client sample](/samples/camel-quarkus-cxf-soap-client/) demonstrates direct route triggering with `camel().send()`.
- **Server-side SOAP testing**: If your application exposes a SOAP service rather than consuming one, the [CXF SOAP server sample](/samples/camel-quarkus-cxf-soap/) demonstrates Citrus as a SOAP client.
- **OpenAPI-driven REST testing**: If your REST API has an OpenAPI specification, the [OpenAPI contract testing sample](/samples/camel-quarkus-openapi-server/) demonstrates specification-driven testing.

To explore the full example project including both test classes and the WSDL contract, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers HTTP endpoints, SOAP WebService endpoints, JSON and XML validation, and the many other transports that Citrus supports.
