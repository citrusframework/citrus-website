---
layout: sample
title: Apache Camel Quarkus - CXF SOAP Client Testing
name: camel-cxf-soap-client
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel CXF SOAP client routes in Quarkus with Citrus service virtualization
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-cxf-soap-client/
---

Many Camel routes do not expose a SOAP service — they consume one. Your application calls an external SOAP WebService as part of a larger integration flow: look up a customer, submit an order, retrieve pricing data. In production, that external service is a real system managed by another team or a third-party provider. In your tests, it needs to be something you control.

This is the opposite of the [CXF SOAP WebService testing sample](/samples/camel-quarkus-cxf-soap/) on this site, where Citrus acts as a SOAP client calling a Camel-hosted service. Here, the roles are reversed: the Camel application is the SOAP client, and Citrus provides a simulated SOAP server that stands in for the external service. The test validates the requests your application sends, controls the responses it receives, and verifies how your application handles both normal responses and SOAP faults.

This post walks you through testing Apache Camel CXF SOAP client routes in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus starts a simulated SOAP server, validates the SOAP requests that the Camel route sends, provides controlled responses, and simulates SOAP faults to verify error handling.

By the end you will have a recipe for testing any Camel route that calls an external SOAP WebService — without depending on the real service.

# The application under test

The example application uses Apache Camel's CXF component as a SOAP client to call an external FruitService. Three Camel routes expose the service operations through `direct:` endpoints that your application code (or tests) can invoke:

```
direct:listFruits   --> CXF client --> external SOAP service (listFruits)
direct:addFruit     --> CXF client --> external SOAP service (addFruit)
direct:deleteFruit  --> CXF client --> external SOAP service (deleteFruit)
```

The WSDL contract is the same one used in the server example — a FruitService with `Fruit` objects that have `name` and `description` properties. The project uses **WSDL2Java code generation** to produce all model objects and the service interface from the WSDL:

```properties
quarkus.cxf.codegen.wsdl2java.includes=wsdl/FruitService.wsdl
```

## The CXF client routes

The `FruitClientRoutes` class configures the CXF endpoint as a producer (client) and defines routes that call the external service:

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

There are a few important details here.

The `CxfEndpoint` is used as a **producer** — the `to("cxf:bean:fruitEndpoint")` directive sends SOAP requests to the external service rather than exposing one. In the server example, the same component appears in a `from()` clause. Here it appears in `to()`, which inverts the direction entirely.

Each route sets the `CxfConstants.OPERATION_NAME` header before calling the CXF endpoint. This header tells CXF which WSDL operation to invoke — `listFruits`, `addFruit`, or `deleteFruit`. In POJO data format mode (the default), CXF marshals the Java objects passed into the route into SOAP XML, sends the request, and unmarshals the response back to Java objects.

The critical detail for testing is the **configurable service URL**. The `fruitServiceUrl` is injected from the `fruit.service.url` property:

```properties
fruit.service.url=http://localhost:18080/services/fruits
```

In tests, this property points to the Citrus SOAP server instead of a real external service. The application code does not change — only the configuration does.

# Adding Citrus to the project

This example needs Citrus modules for SOAP server simulation, Camel route triggering, and XML validation:

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
    <artifactId>citrus-junit-jupiter</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Here is what each module brings to the table:

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle.
- **citrus-camel** provides the `camel()` DSL for triggering Camel routes and verifying exchange results — including exception handling.
- **citrus-ws** provides the `WebServiceServer` that simulates the external SOAP service, plus the `soap()` DSL for server-side request validation and response simulation.
- **citrus-validation-xml** enables structural XML validation and JAXB unmarshalling for incoming SOAP requests.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

The key module is **citrus-ws** in server mode. In the [server testing sample](/samples/camel-quarkus-cxf-soap/), Citrus used a `WebServiceClient` to call a Camel-hosted SOAP service. Here, Citrus uses a `WebServiceServer` to receive calls from the Camel CXF client. Same module, opposite role.

# Setting up the test class

The test class wires together the Citrus SOAP server, the shared CamelContext, and the test runner:

```java
@QuarkusTest
@CitrusSupport
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class FruitSoapClientTest implements TestActionSupport {

    @CitrusEndpoint
    @WebServiceServerConfig(port = 18080, autoStart = true,
            timeout = 10000)
    WebServiceServer soapServer;

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

The `@WebServiceServerConfig` annotation starts a Citrus SOAP server on port 18080 — the same port that `fruit.service.url` points to in `application.properties`. When the Camel CXF client sends a SOAP request to `http://localhost:18080/services/fruits`, it reaches this simulated server instead of a real external service.

The `autoStart = true` setting ensures the server is ready before any test method runs. The `timeout = 10000` gives the server 10 seconds to receive an expected request before failing — enough time for Quarkus startup and CXF initialization.

The `@TestInstance(TestInstance.Lifecycle.PER_CLASS)` annotation shares the server instance across all test methods. Without this, JUnit would create a new test instance (and try to bind a new server to the same port) for each test, causing port conflicts.

The `CamelContext` is injected with `@Inject` and registered in the Citrus registry with `@BindToRegistry`. This enables the `camel()` DSL actions to send messages into the application's Camel routes and receive exchange results — including exceptions.

# Triggering routes and validating requests

The core testing pattern has three steps: trigger the Camel route, validate the SOAP request on the server side, and send back a simulated response.

## Listing fruits

The simplest test triggers a route with no input and validates the SOAP request structure:

```java
@Test
void shouldListFruits() {
    runner.when(
        async().actions(
            action(context -> {
                ProducerTemplate template =
                        camelContext.createProducerTemplate();
                template.requestBody("direct:listFruits",
                        (Object) null);
            })
        )
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

The `async().actions(...)` block triggers the Camel route in a separate thread. This is essential because the CXF client call is synchronous — the route blocks until it receives a response from the SOAP server. Without the async wrapper, the test thread would be stuck in the route and never reach the server-side receive and send steps.

The `soap().server(soapServer).receive()` action waits for the CXF client's SOAP request and validates the body. Citrus strips the SOAP envelope and compares just the body content. The `soap().server(soapServer).send()` action sends back the simulated response, which CXF unmarshals and returns to the Camel route.

## Adding a fruit with camel().send()

The second test demonstrates a cleaner approach using the Citrus Camel component:

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
                "<fruits>" +
                    "<fruit>...</fruit>" +
                    "<fruit>...</fruit>" +
                    "<fruit>" +
                        "<description>Tropical fruit</description>" +
                        "<name>Mango</name>" +
                    "</fruit>" +
                "</fruits>" +
            "</ns:addFruitResponse>")
    );
}
```

Instead of creating a `ProducerTemplate` manually, `camel().send().endpoint("direct:addFruit")` sends a message into the route using the Citrus Camel DSL. The `Fruit` POJO is wrapped in a `DefaultMessage` and passed as the message body. The `fork(true)` option serves the same purpose as `async()` — it runs the route call in a separate thread so the test can proceed to the server-side steps.

The server-side validation confirms that CXF correctly marshalled the Java `Fruit` object into the expected SOAP XML structure — element names, namespace, and content all match.

# Simulating SOAP faults

The most interesting test verifies how the application handles errors from the external service. When the CXF client receives a SOAP fault, it translates it into a Java exception on the Camel exchange. Citrus can both simulate the fault and verify the resulting exception.

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
        camel().receive()
            .endpoint("direct:deleteFruit", true)
            .message()
            .validate((message, context) -> {
                Assertions.assertNotNull(
                    message.getHeader(
                        CamelMessageHeaders.EXCHANGE_EXCEPTION));
                Assertions.assertEquals(
                    "Fruit \"Pineapple\" does not exist.",
                    message.getHeader(
                        CamelMessageHeaders.EXCHANGE_EXCEPTION_MESSAGE));
            })
    );
}
```

This test has four steps, each covering a different aspect of the fault handling flow.

**Step 1 — trigger the route with exception capture.** The `camel().send().endpoint("direct:deleteFruit", true)` call has a second parameter `true` that enables exception capture on the endpoint. Without this flag, an exception on the Camel exchange would propagate immediately and fail the test. With it, Citrus captures the exception so you can inspect it later.

**Step 2 — validate the outgoing request.** The SOAP server receives the delete request and confirms that the CXF client sent the correct Pineapple fruit in the SOAP body.

**Step 3 — simulate the SOAP fault.** The `sendFault()` action sends a SOAP fault response instead of a normal response. The `faultCode` specifies the SOAP fault code (a `Server` fault in the SOAP namespace), and `faultString` provides the human-readable error message. CXF on the client side translates this SOAP fault into a Java exception on the Camel exchange.

**Step 4 — verify the captured exception.** The `camel().receive().endpoint("direct:deleteFruit", true)` action retrieves the completed Camel exchange and validates the captured exception. The `CamelMessageHeaders.EXCHANGE_EXCEPTION` header contains the exception class, and `CamelMessageHeaders.EXCHANGE_EXCEPTION_MESSAGE` contains the fault string. This confirms that the SOAP fault was correctly translated into a Java exception that your application code can handle.

# JAXB-based server validation

The example also includes a POJO-based test class that uses JAXB marshalling on the server side. Instead of comparing XML strings, the server unmarshals the incoming SOAP request into typed Java objects for programmatic validation.

## Setup with marshaller and test profile

```java
@QuarkusTest
@CitrusSupport
@TestProfile(FruitPojoSoapClientTest.TestConfig.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class FruitPojoSoapClientTest implements TestActionSupport {

    @CitrusEndpoint
    @WebServiceServerConfig(port = 18081, autoStart = true,
            timeout = 10000)
    WebServiceServer soapServer;

    @BindToRegistry
    public Jaxb2Marshaller marshaller() {
        return new Jaxb2Marshaller(Fruit.class.getPackageName());
    }

    public static class TestConfig implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("fruit.service.url",
                    "http://localhost:18081/services/fruits");
        }
    }
}
```

This test runs on port 18081 (not 18080) to avoid conflicts with the XML-based test class. The `@TestProfile` overrides the `fruit.service.url` property to point the CXF client at this server instead.

## Validating requests as Java objects

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

The `XmlMarshallingValidationProcessor` automatically unmarshals the incoming SOAP request body into a typed `JAXBElement<AddFruit>`. You access the request fields through Java getters instead of XML comparison — if a field name changes in the WSDL and the generated types are regenerated, the compiler catches the mismatch.

## Sending responses as Java objects

Responses are constructed from the generated types and marshalled to XML:

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

The `marshal()` helper converts the Java response object to XML using the registered `Jaxb2Marshaller`. Helper methods like `fruit()` and `addFruitResponse()` keep the test code concise — they construct the generated WSDL2Java types and wrap them in the proper `JAXBElement` using the `ObjectFactory`.

# How it all fits together

When you run the tests with `./mvnw clean test`, here is the sequence of events:

1. **WSDL2Java generates** all model objects and the service interface from `FruitService.wsdl`.
2. **Citrus starts** the SOAP server on port 18080 (or 18081 for the POJO test).
3. **Quarkus starts** the application with `fruit.service.url` pointing to the Citrus server.
4. **Apache Camel discovers** the CXF client routes and configures the endpoint.
5. **The test triggers** a Camel route using `camel().send()` or `ProducerTemplate`.
6. **The CXF client** marshals the request to SOAP XML and sends it to the Citrus server.
7. **Citrus validates** the incoming SOAP request (XML comparison or JAXB unmarshalling).
8. **Citrus sends** a simulated response (or a SOAP fault).
9. **The CXF client** unmarshals the response and returns it to the Camel route.
10. **The test verifies** the exchange result (or the captured exception for fault scenarios).

The entire test suite runs in under five seconds with no external services. Seven tests across two test classes verify all three operations plus SOAP fault handling using both XML and JAXB approaches.

# Server testing vs client testing

This example and the [CXF SOAP server sample](/samples/camel-quarkus-cxf-soap/) are two sides of the same coin:

| Aspect | Server testing | Client testing |
|--------|---------------|---------------|
| Camel CXF role | Receives SOAP requests | Sends SOAP requests |
| Citrus role | SOAP client | SOAP server |
| What you validate | Response your service produces | Request your route sends |
| What you simulate | — | Response from external service |
| Fault testing | `assertFault()` on receive | `sendFault()` on server |

Many real-world applications do both — they expose a SOAP service while also calling other SOAP services. You can combine both testing patterns in the same project.

# Where to go from here

This example covers the core client-side SOAP testing patterns. Here are several directions you can take it:

- **Error handling routes**: Add a Camel `onException()` handler that catches SOAP faults and routes them to a dead-letter endpoint or an error response. Verify the error flow end to end.
- **Response transformation**: Add Camel route logic that transforms the SOAP response before passing it downstream — for example, mapping `Fruit` objects to a different domain model. Test the transformation with the simulated responses.
- **Timeout testing**: Configure a delay in the Citrus server response and verify that the CXF client's timeout handling works correctly.
- **Multiple external services**: If your application calls several SOAP services, set up multiple Citrus `WebServiceServer` instances on different ports and test the complete orchestration.
- **REST-to-SOAP bridge**: If your application exposes a REST API that internally calls a SOAP service, combine the [REST DSL sample](/samples/camel-quarkus-rest-dsl/) patterns with this SOAP client testing approach.
- **Server-side testing**: If you need to test a Camel-hosted SOAP service instead, the [CXF SOAP server sample](/samples/camel-quarkus-cxf-soap/) demonstrates Citrus as a SOAP client.

To explore the full example project including the WSDL contract and both test classes, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers SOAP WebService endpoints, Camel integration, XML validation, and the many other transports that Citrus supports.
