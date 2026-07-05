---
layout: sample
title: Apache Camel Quarkus - REST DSL Testing
name: camel-rest-dsl
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel REST DSL endpoints in Quarkus with Citrus HTTP client
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-rest-dsl/
---

Apache Camel's REST DSL gives you a clean, declarative way to define REST APIs in Quarkus. You map HTTP verbs and paths to Camel routes, keep the API definition separate from the business logic, and let Camel handle JSON serialization, content negotiation, and error responses.

But once the endpoints are up and running, you need to verify them. Does a POST request return 201 CREATED as a response code as expected? Does the DELETE operation of a missing resource return 404 NOT FOUND error? Does the GET response contain the right JSON structure? 

These are the questions that integration tests should answer — with a real HTTP server, real requests, and real response validation.

This post walks you through testing an Apache Camel REST DSL service in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus acts as an HTTP client to exercise every endpoint, validate status codes and JSON bodies, and even deserialize responses into typed Java objects for assertion. 

By the end you will have a clear recipe for testing your own Camel REST APIs in Quarkus.

# The application under test

The example application implements a simple fruits REST API using Apache Camel REST DSL. It supports three operations: listing all fruits, adding a new fruit, and deleting a fruit by name.

```
GET    /fruits  ->  Returns list of all fruits     (200)
POST   /fruits  ->  Creates a new fruit            (201)
DELETE /fruits  ->  Deletes a fruit by name   (204 or 404)
```

The Camel REST DSL definition maps each HTTP verb to an internal `direct:` route:

```java
rest("/fruits")
        .get()
            .produces(APPLICATION_JSON_CONTENT_TYPE)
            .to("direct:get-fruits")
        .post()
            .consumes(APPLICATION_JSON_CONTENT_TYPE)
            .to("direct:create-fruit")
        .delete()
            .consumes(APPLICATION_JSON_CONTENT_TYPE)
            .to("direct:delete-fruit");
```

This is one of the strengths of REST DSL — the API contract is defined in one place, cleanly separated from the processing logic. Each verb delegates to a `direct:` endpoint, which is Camel's in-memory routing mechanism for connecting routes without any transport overhead.

The processing routes handle JSON marshalling and business logic:

```java
from("direct:get-fruits")
        .setBody(exchange -> new ArrayList<>(fruits))
        .marshal().json();

from("direct:create-fruit")
        .unmarshal().json(Fruit.class)
        .process(exchange -> {
            Fruit fruit = exchange.getIn().getBody(Fruit.class);
            fruits.add(fruit);
            exchange.getIn().setBody(null);
            exchange.getIn().setHeader(Exchange.HTTP_RESPONSE_CODE, 201);
        });

from("direct:delete-fruit")
        .unmarshal().json(Fruit.class)
        .process(exchange -> {
            Fruit fruit = exchange.getIn().getBody(Fruit.class);
            boolean found = fruits.removeIf(f -> f.getName().equals(fruit.getName()));
            exchange.getIn().setBody(null);
            exchange.getIn().setHeader(Exchange.HTTP_RESPONSE_CODE, !found ? 404 : 204);
        });
```

The get route converts the in-memory fruit list to JSON using Camel's Jackson data format. The create route deserializes the incoming JSON into a `Fruit` object, adds it to the store, and returns 201. The delete route looks up the fruit by name, removes it if found and returns 204, or returns 404 if it does not exist.

The fruits are stored in a simple `LinkedList` seeded with Apple, Mango, and Orange as initial data. Quarkus automatically discovers the `RouteBuilder`, starts the Camel context, and serves the REST endpoints on its built-in HTTP server. That is the entire application — no JAX-RS resources, no controller classes, just Camel routes.

# Adding Citrus to the project

For testing REST endpoints, Citrus needs its HTTP client module and JSON validation support. Add these test-scoped dependencies to the Maven POM:

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
- **citrus-http** provides the HTTP client endpoint that sends requests to your REST API and receives responses.
- **citrus-validation-json** adds JSON-aware message validation, including JsonPath expressions and validation matchers.
- **citrus-junit-jupiter** connects Citrus to JUnit Jupiter.

Unlike the messaging examples on this site, there is no external broker to provision. The Quarkus test framework starts the application with its built-in HTTP server, and Citrus connects to it directly.

# Setting up the test class

The test class needs just two annotations and an HTTP client endpoint:

```java
@QuarkusTest
@CitrusSupport
class FruitRestDslTest implements TestActionSupport {

    @CitrusEndpoint
    @HttpClientConfig(requestUrl = "http://localhost:8081")
    HttpClient fruitRestClient;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

`@QuarkusTest` starts the application in test mode on port 8081. `@CitrusSupport` layers Citrus on top of the Quarkus lifecycle. The `@HttpClientConfig` annotation configures a Citrus HTTP client pointing to the Quarkus test server — this client is reused across all test methods to send requests and validate responses.

There is no connection factory to share, no broker to provision, and no container to start. The Quarkus HTTP server is the only infrastructure, and it is managed automatically by the test framework.

# Testing the CRUD operations

With the client configured, each test follows a simple pattern: send an HTTP request in the **when** block, validate the response in the **then** block.

## Creating a fruit

```java
@Test
void shouldAddFruit() {
    runner.when(
            http()
                .client(fruitRestClient)
                .send()
                .post()
                .path("/fruits")
                .message()
                .body("""
                {
                   "name": "Pineapple",
                   "description": "A sweet and tasty tropical fruit."
                }
                """)
                .contentType(APPLICATION_JSON_CONTENT_TYPE)
    );

    runner.then(
            http()
                .client(fruitRestClient)
                .receive()
                .response(201)
    );
}
```

The test sends a POST request with a JSON body containing the new fruit. The `response(201)` assertion verifies that the Camel route correctly set the HTTP status code to 201 Created. If the route returned any other status code, Citrus would fail the test immediately with a clear message showing the expected vs actual status.

## Listing all fruits

```java
@Test
void shouldListFruits() {
    runner.when(
            http()
                .client(fruitRestClient)
                .send()
                .get()
                .path("/fruits")
    );

    runner.then(
            http()
                .client(fruitRestClient)
                .receive()
                .response(200)
                .message()
                .validate(validation().jsonPath()
                        .expression("$..name", "@contains(Apple,Mango,Orange)@"))
    );
}
```

This test introduces JsonPath validation. Instead of comparing the entire JSON body, the `validation().jsonPath()` builder lets you target specific elements. The expression `$..name` selects all `name` fields in the response array, and the `@contains()@` validation matcher verifies that the result includes Apple, Mango, and Orange — without requiring a specific order or exact match on other fields.

This flexible validation is essential for REST API testing. You often care about whether certain elements are present rather than whether the entire response matches byte for byte. JsonPath expressions give you that precision.

## Deleting a fruit

```java
@Test
void shouldDeleteFruit() {
    runner.when(
            http()
                .client(fruitRestClient)
                .send()
                .delete("/fruits")
                .message()
                .contentType(APPLICATION_JSON_CONTENT_TYPE)
                .body("""
                {
                    "name": "Apple"
                }
                """)
    );

    runner.then(
            http()
                .client(fruitRestClient)
                .receive()
                .response(204)
    );
}
```

The DELETE test sends a JSON body identifying the fruit to remove and verifies that the route returns 204 No Content — the standard HTTP response for a successful deletion with no response body.

## Handling the not-found case

```java
@Test
void shouldHandleFruitNotFound() {
    runner.when(
            http()
                .client(fruitRestClient)
                .send()
                .delete("/fruits")
                .message()
                .contentType(APPLICATION_JSON_CONTENT_TYPE)
                .body("""
                {
                    "name": "Pineapple",
                    "description": "A sweet and tasty tropical fruit."
                }
                """)
    );

    runner.then(
            http()
                .client(fruitRestClient)
                .receive()
                .response(404)
    );
}
```

Error cases are just as important as happy paths. This test attempts to delete a fruit that does not exist in the store and verifies that the Camel route returns 404 Not Found. Testing error responses ensures your API behaves correctly when clients send invalid requests — and with Citrus, it takes exactly the same amount of code as testing the success case.

# POJO-based testing

The tests above use inline JSON strings for request bodies. This works well for simple cases, but as your API grows, maintaining JSON strings across many tests becomes tedious. Citrus offers an alternative: POJO-based testing with automatic JSON marshalling.

First, register a Jackson `ObjectMapper` with the Citrus registry:

```java
@BindToRegistry
ObjectMapper objectMapper = JsonMapper.builder()
        .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
        .enable(SerializationFeature.INDENT_OUTPUT)
        .build();
```

Now you can use Java objects directly as request bodies with the `marshal()` helper:

```java
runner.when(
        http()
            .client(fruitRestClient)
            .send()
            .post()
            .path("/fruits")
            .message()
            .body(marshal(new Fruit("Pineapple", "A sweet and tasty tropical fruit.")))
            .contentType(APPLICATION_JSON_CONTENT_TYPE)
);
```

Instead of writing a JSON string, you construct a `Fruit` object and let Citrus serialize it. This approach gives you compile-time safety — if the `Fruit` constructor changes, the test fails at compile time rather than at runtime with a confusing JSON parse error.

The real power shows on the response side. Use `JsonMappingValidationProcessor` to deserialize the response into a typed array and validate it with standard Java assertions:

```java
runner.then(
        http()
            .client(fruitRestClient)
            .receive()
            .response(200)
            .message()
            .validate(new JsonMappingValidationProcessor<>(Fruit[].class) {
                @Override
                public void validate(Fruit[] fruits, Map<String, Object> headers,
                                     TestContext context) {
                    Assertions.assertTrue(fruits.length > 0);
                    Assertions.assertTrue(Arrays.stream(fruits)
                            .anyMatch(fruit -> fruit.getName().equals("Apple")));
                }
            })
);
```

The processor deserializes the JSON response into a `Fruit[]` array, giving you full type safety and the ability to use any Java assertion logic. You can check array lengths, filter by properties, compare objects, or run any validation that Java supports. This is especially useful for complex response structures where JsonPath expressions would become unwieldy.

# How it all fits together

When you run the tests with `./mvnw clean test`, here is the sequence of events:

1. **Quarkus starts** the application in test mode on port 8081.
2. **Apache Camel discovers** the `RouteBuilder` and registers the REST DSL endpoints.
3. **Citrus initializes** the HTTP client pointing to the Quarkus test server.
4. **Each test sends** an HTTP request (GET, POST, or DELETE) through the Citrus client.
5. **The Camel REST DSL** routes the request to the matching `direct:` route for processing.
6. **The processing route** handles JSON marshalling, business logic, and sets the HTTP status code.
7. **Citrus receives** the response and validates the status code, headers, and body.
8. **The application shuts down** after all tests complete.

Because Citrus and Quarkus share the same test lifecycle, there is no external infrastructure to manage. The Quarkus HTTP server starts and stops automatically, and the Citrus HTTP client connects directly to it. This makes REST DSL tests fast to run and simple to maintain.

As with all Citrus test classes, these are standard JUnit Jupiter tests. You can run them from your favorite Java IDE, in a CI pipeline, or from the command line with Maven.

# Where to go from here

This example covers the fundamentals of REST API testing with Citrus, but there is much more you can do:

- **Request and response headers**: Citrus lets you set custom HTTP headers on requests and validate them on responses. This is useful for testing authentication, content negotiation, and caching behavior.
- **XML validation**: If your Camel REST endpoints produce XML instead of JSON, add the `citrus-validation-xml` module for XPath-based validation with the same fluent API.
- **Complex Camel patterns**: Test content-based routing, message transformation pipelines, and error handling. Camel REST DSL routes can delegate to arbitrarily complex route chains, and Citrus can validate the final HTTP response regardless of what happens internally.
- **OpenAPI contract testing**: The example application includes an OpenAPI specification. You can use it to generate test cases or validate that your Camel routes conform to the API contract.
- **Messaging comparison**: If your Camel routes use JMS or Kafka instead of HTTP, the [Camel JMS sample](/samples/camel-quarkus-jms/) and [Camel Kafka sample](/samples/camel-quarkus-kafka/) on this site demonstrate the equivalent setup with messaging endpoints.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers HTTP endpoints, JSON validation, Quarkus integration, and the many other transports that Citrus supports.
