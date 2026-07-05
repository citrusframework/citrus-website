---
layout: sample
title: Apache Camel Quarkus - CXF SOAP WebService Testing
name: camel-cxf-soap
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel CXF SOAP WebServices in Quarkus with Citrus SOAP client and JAXB marshalling
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-cxf-soap/
---

SOAP WebServices are still a reality in enterprise integration. Contract-first, WSDL-driven services power banking systems, healthcare exchanges, government APIs, and countless internal platforms. Apache Camel's CXF component brings these services into the Camel routing world, handling the SOAP envelope processing, JAXB marshalling, and operation dispatching so you can focus on business logic.

Testing SOAP services presents unique challenges. You need to construct valid SOAP envelopes, match the WSDL contract, validate structured XML responses, and handle SOAP faults — all on top of the usual integration test setup. Getting the namespace declarations wrong, missing a required element, or mismatching a fault code can turn debugging into an exercise in XML archaeology.

This post walks you through testing an Apache Camel CXF SOAP WebService in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see two complementary approaches: testing with raw XML bodies for full visibility into the SOAP messages, and testing with JAXB-marshalled Java objects for type-safe validation. Both approaches use Citrus's `soap()` DSL, which handles SOAP envelope wrapping automatically and provides dedicated support for SOAP fault assertions.

By the end you will have a recipe for testing WSDL-based services with Citrus — from happy-path operations to error scenarios with SOAP faults.

# The application under test

The example application exposes a FruitService SOAP API. It manages a simple in-memory collection of fruits with three operations: list all fruits, add a new fruit, and delete an existing fruit. The entire service is contract-first — the WSDL defines the operations, types, and faults, and the code is generated from it.

## The WSDL contract

The service contract is defined in `FruitService.wsdl` with the target namespace `http://camel.apache.org/test/FruitService`. It defines a `Fruit` type with `name` and `description` properties, three operations, and a fault:

| Operation | Input | Output | Fault |
|-----------|-------|--------|-------|
| `listFruits` | (none) | List of fruits | — |
| `addFruit` | A fruit to add | Updated list of fruits | — |
| `deleteFruit` | A fruit to delete | Remaining fruits | `NoSuchFruitException` |

The project uses Quarkus CXF's **WSDL2Java code generation** to produce all model objects, the service endpoint interface, and fault classes from the WSDL. A single line in `application.properties` drives the generation:

```properties
quarkus.cxf.codegen.wsdl2java.includes=wsdl/FruitService.wsdl
```

This generates classes like `Fruit`, `FruitService`, `AddFruit`, `AddFruitResponse`, `ListFruitsResponse`, `NoSuchFruitException`, and an `ObjectFactory` for creating JAXB elements — all in the `org.apache.camel.test.fruitservice` package.

## The service implementation

The `FruitServiceImpl` class implements the generated `FruitService` interface as a CDI bean:

```java
@ApplicationScoped
@Named("fruitService")
public class FruitServiceImpl implements FruitService {

    private final List<Fruit> fruits = new ArrayList<>();

    public FruitServiceImpl() {
        Fruit apple = new Fruit();
        apple.setName("Apple");
        apple.setDescription("Winter fruit");
        fruits.add(apple);

        Fruit orange = new Fruit();
        orange.setName("Orange");
        orange.setDescription("Citrus fruit");
        fruits.add(orange);
    }

    @Override
    public AddFruitResponse.Fruits addFruit(Fruit fruit) {
        fruits.add(fruit);
        AddFruitResponse.Fruits response = new AddFruitResponse.Fruits();
        response.getFruit().addAll(fruits);
        return response;
    }

    @Override
    public DeleteFruitResponse.Fruits deleteFruit(Fruit fruit)
            throws NoSuchFruitException {
        // ... find and remove the fruit, or throw NoSuchFruitException
    }

    @Override
    public ListFruitsResponse.Fruits listFruits() {
        ListFruitsResponse.Fruits response = new ListFruitsResponse.Fruits();
        response.getFruit().addAll(fruits);
        return response;
    }
}
```

The service operates on an in-memory `ArrayList` seeded with Apple and Orange. Each method works with the generated JAXB types directly — the `@Named("fruitService")` annotation makes the bean available for Camel's bean component to invoke by name.

The `deleteFruit` method is the interesting one for testing: it throws `NoSuchFruitException` when you try to delete a fruit that does not exist. CXF translates this into a SOAP fault response automatically, following the fault definition in the WSDL.

## The Camel CXF route

The `FruitServiceRoutes` class wires the CXF SOAP endpoint to the service bean:

```java
public class FruitServiceRoutes extends RouteBuilder {

    @Produces
    @ApplicationScoped
    @Named
    CxfEndpoint fruitEndpoint() {
        CxfEndpoint fruitEndpoint = new CxfEndpoint();
        fruitEndpoint.setServiceClass(FruitService.class);
        fruitEndpoint.setAddress("/fruits");
        return fruitEndpoint;
    }

    @Override
    public void configure() throws Exception {
        from("cxf:bean:fruitEndpoint")
            .recipientList(
                simple("bean:fruitService?method=${header.operationName}"));
    }
}
```

The route is a single line, but it does a lot. The `CxfEndpoint` CDI producer creates a named bean that configures the CXF service with the generated `FruitService` interface and the `/fruits` address. Combined with the `quarkus.cxf.path=/cxf/services` property, the full endpoint URL becomes `http://localhost:8080/cxf/services/fruits`.

The route itself uses Camel's **Recipient List** pattern with a Simple expression. When a SOAP request arrives, CXF sets the `operationName` header to the WSDL operation name (e.g., `addFruit`, `deleteFruit`, `listFruits`). The recipient list dispatches the message to the matching method on the `fruitService` bean. In POJO data format mode (the default), CXF handles all the JAXB marshalling and unmarshalling between SOAP XML and Java objects — the route works with typed objects, not raw XML.

# Adding Citrus to the project

This example uses Citrus's WebService module for SOAP client support and XML validation for response checking:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-quarkus</artifactId>
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
- **citrus-ws** provides the SOAP WebService client with `soap()` DSL actions, automatic SOAP envelope handling, and SOAP fault assertion support.
- **citrus-validation-xml** enables structural XML validation for SOAP response bodies and supports JAXB-based marshalling validation.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

The key module is **citrus-ws**. It wraps your message body in a proper SOAP envelope before sending, strips the envelope on receive, and provides the `assertFault()` action for verifying SOAP fault responses — all the SOAP plumbing that would otherwise clutter your tests.

# Approach 1: XML-based testing

The first approach uses raw XML strings for request bodies and response validation. This gives you full visibility into the SOAP message content and is the most direct way to verify the XML structure.

## Setting up the test class

```java
@QuarkusTest
@CitrusSupport
public class FruitSoapServiceTest implements TestActionSupport {

    @CitrusEndpoint
    @WebServiceClientConfig(
            requestUrl = "http://localhost:8081/cxf/services/fruits")
    WebServiceClient soapClient;

    @CitrusResource
    GherkinTestActionRunner runner;

    @BindToRegistry
    public SoapMessageFactory messageFactory() {
        SaajSoapMessageFactory factory = new SaajSoapMessageFactory();
        factory.afterPropertiesSet();
        return factory;
    }
}
```

The `@WebServiceClientConfig` annotation configures a Citrus SOAP client pointing at the CXF endpoint on the Quarkus test port. The `SoapMessageFactory` registered with `@BindToRegistry` provides the SOAP message implementation that the client uses to create and parse SOAP envelopes — you register it once and forget about it.

## Listing fruits

The first test verifies the initial state of the service:

```java
@Test
void shouldListFruits() {
    runner.when(
        soap().client(soapClient)
            .send()
            .message()
            .soapAction("")
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

The `soap().client(soapClient).send()` action sends a SOAP request. You provide only the SOAP body — Citrus wraps it in a full SOAP envelope automatically. The `soapAction("")` sets the SOAPAction HTTP header, which matches the empty action defined in the WSDL binding.

The `soap().client(soapClient).receive()` action receives the SOAP response and validates the body against the expected XML. Citrus strips the SOAP envelope and compares the body element structurally — element names, namespace URIs, text content, and nesting must all match.

## Adding a fruit

```java
@Test
void shouldAddFruit() {
    runner.when(
        soap().client(soapClient)
            .send()
            .message()
            .soapAction("")
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

The request wraps a `Fruit` element inside an `addFruit` operation element. The response validation confirms that the service returned the updated fruit list including the newly added Mango.

## Handling SOAP faults

The most interesting test verifies error handling. When you try to delete a fruit that does not exist, the service throws a `NoSuchFruitException`, which CXF translates into a SOAP fault:

```java
@Test
void shouldHandleFruitNotFound() {
    runner.then(
        soap().client(soapClient)
            .assertFault()
            .faultCode(
                "{http://schemas.xmlsoap.org/soap/envelope/}Server")
            .faultString(
                "Fruit \"Pineapple\" does not exist.")
            .when(soap().client(soapClient)
                .send()
                .message()
                .soapAction("")
                .body("<ns:deleteFruit xmlns:ns=\"" + TARGET_NS + "\">" +
                    "<fruit>" +
                        "<description>Tropical fruit</description>" +
                        "<name>Pineapple</name>" +
                    "</fruit>" +
                "</ns:deleteFruit>"))
    );
}
```

The `assertFault()` action is Citrus's dedicated SOAP fault verifier. It wraps a send action and asserts that the response is a SOAP fault rather than a normal response. The `faultCode` check verifies the SOAP fault code — `{http://schemas.xmlsoap.org/soap/envelope/}Server` indicates a server-side error. The `faultString` check verifies the human-readable error message.

Notice the structural difference from the other tests. The send action is nested inside the `assertFault().when(...)` call rather than being a separate step. This is because SOAP faults are protocol-level errors, not regular responses — Citrus needs to intercept the fault before normal response processing would treat it as an unexpected error.

# Approach 2: JAXB-based testing

The second approach uses the WSDL2Java generated model objects for both request construction and response validation. This provides compile-time type safety and a more natural Java programming style.

## Setting up JAXB marshalling

The POJO test registers a `Jaxb2Marshaller` that knows how to convert between Java objects and XML:

```java
@BindToRegistry
public Jaxb2Marshaller marshaller() {
    return new Jaxb2Marshaller(Fruit.class.getPackageName());
}
```

The marshaller is initialized with the package name of the generated classes. It discovers all JAXB-annotated types in that package and can marshal any of them to XML and unmarshal XML back to typed objects.

## Sending requests as Java objects

Instead of writing XML strings, you construct request objects using the generated types:

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
        .soapAction("")
        .body(marshal(new ObjectFactory().createAddFruit(addFruit)))
);
```

The `marshal()` helper converts the Java object to XML using the registered `Jaxb2Marshaller`. The `ObjectFactory` — generated by WSDL2Java — creates a properly typed `JAXBElement` wrapper that ensures the correct XML element name and namespace. You work with Java objects throughout; the JAXB layer handles the XML serialization.

## Validating responses as Java objects

Response validation uses `XmlMarshallingValidationProcessor`, which automatically unmarshals the SOAP response body into a typed Java object:

```java
runner.then(
    soap().client(soapClient)
        .receive()
        .message()
        .validate(
            new XmlMarshallingValidationProcessor
                    <JAXBElement<AddFruitResponse>>() {
                @Override
                public void validate(
                        JAXBElement<AddFruitResponse> payload,
                        Map<String, Object> headers,
                        TestContext context) {
                    AddFruitResponse response = payload.getValue();
                    Assertions.assertTrue(
                        response.getFruits().getFruit().stream()
                            .anyMatch(fruit ->
                                "Mango".equals(fruit.getName())));
                }
            })
);
```

The processor receives the unmarshalled `JAXBElement<AddFruitResponse>` directly. From there, you use standard Java assertions — stream operations, equality checks, collection sizes — to validate the response content. If a field is renamed in the WSDL and the generated types change, the compiler catches the mismatch immediately instead of failing at runtime with an XML comparison error.

## SOAP faults with JAXB

The SOAP fault test works the same way with marshalled objects:

```java
Fruit pineapple = new Fruit();
pineapple.setName("Pineapple");
pineapple.setDescription("Tropical fruit");

DeleteFruit deleteFruit = new DeleteFruit();
deleteFruit.setFruit(pineapple);

runner.then(
    soap().client(soapClient)
        .assertFault()
        .faultCode("{http://schemas.xmlsoap.org/soap/envelope/}Server")
        .faultString("Fruit \"Pineapple\" does not exist.")
        .when(soap().client(soapClient)
            .send()
            .message()
            .soapAction("")
            .body(marshal(
                new ObjectFactory().createDeleteFruit(deleteFruit))))
);
```

The request is constructed from Java objects, but the fault assertion uses the same `faultCode` and `faultString` checks. SOAP faults are protocol-level structures that do not map to JAXB types in the same way as regular responses, so the assertion API stays the same across both approaches.

# XML vs JAXB — when to use which

Both approaches test the same service and cover the same scenarios. The choice depends on what you want to optimize for:

**Use XML-based testing when:**
- You want full visibility into the exact XML being sent and received
- You are debugging a SOAP interoperability issue where element order or namespace prefixes matter
- You are testing a service where you do not have the WSDL2Java generated types (e.g., a third-party service)
- You want the simplest possible test setup with no marshaller configuration

**Use JAXB-based testing when:**
- You want compile-time safety — if the WSDL changes and types are regenerated, the compiler catches mismatches
- You are building complex request objects that would be tedious to write as XML strings
- You want to use programmatic assertions (stream operations, computed comparisons) instead of XML string matching
- Your team is more comfortable reading Java objects than XML structures

Many projects use both: XML-based tests for contract conformance and protocol-level verification, JAXB-based tests for business logic validation with complex data structures.

# How it all fits together

When you run the tests with `./mvnw clean test`, here is the sequence of events:

1. **WSDL2Java generates** all model objects, the service interface, and fault classes from `FruitService.wsdl`.
2. **Quarkus starts** the application in test mode on port 8081.
3. **Apache Camel deploys** the CXF SOAP endpoint at `/cxf/services/fruits` with the `FruitService` interface.
4. **Citrus creates** the SOAP client pointing at the CXF endpoint.
5. **Each test sends** a SOAP request (either as XML or marshalled Java objects).
6. **CXF receives** the request, unmarshals it to Java objects, and sets the `operationName` header.
7. **The Camel route** dispatches to the matching method on the `fruitService` bean.
8. **The service bean** executes the business logic and returns a response (or throws a fault).
9. **CXF marshals** the response back to SOAP XML and sends it to the client.
10. **Citrus validates** the response body or fault against the expected values.

The entire test suite runs in under five seconds with no containers, no external services, and no mock frameworks. Eight tests across two test classes — four with XML bodies, four with JAXB objects — verify all three operations plus SOAP fault handling.

There is one piece of test configuration worth mentioning:

```properties
quarkus.arc.ignored-split-packages=org.citrusframework.*
```

This avoids CDI bean discovery conflicts between Citrus and Quarkus.

# Where to go from here

This example covers the core SOAP testing patterns, but there are several directions you can take it:

- **WS-Security**: Add SOAP message security (username token, digital signature, encryption) and verify that the service enforces authentication correctly.
- **Custom SOAP headers**: Add custom headers to requests and validate that the service processes them — many enterprise SOAP services use headers for correlation IDs, transaction contexts, or routing metadata.
- **Schema validation**: Use `citrus-validation-xml` with the WSDL's embedded XSD schema to validate response structure automatically rather than comparing against expected XML strings.
- **Service virtualization**: If your Camel CXF route calls a downstream SOAP service, use Citrus's `WebServiceServer` to simulate it — the same pattern shown in the [HTTP service virtualization sample](/samples/camel-quarkus-direct-http/), but for SOAP.
- **REST comparison**: If your API is REST-based rather than SOAP, the [OpenAPI contract testing sample](/samples/camel-quarkus-openapi-server/) demonstrates the equivalent specification-driven approach for REST endpoints.
- **Simpler routes**: If your Camel routes do not use SOAP or external services, the [direct endpoint sample](/samples/camel-quarkus-direct/) demonstrates the simplest possible testing setup.

To explore the full example project including the WSDL contract and the complete test code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers SOAP WebService endpoints, XML validation, JAXB marshalling, and the many other transports that Citrus supports.
