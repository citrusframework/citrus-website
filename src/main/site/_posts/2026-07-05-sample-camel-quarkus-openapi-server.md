---
layout: sample
title: Apache Camel Quarkus - OpenAPI Contract Testing
name: camel-openapi-server
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel OpenAPI REST services in Quarkus with specification-driven Citrus test actions
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-openapi-server/
---

REST APIs are everywhere, and most of them start with a contract — an OpenAPI specification that defines the endpoints, request formats, response schemas, and status codes. Apache Camel can generate an entire REST service from that specification with just a few lines of code. The routes, the parameter binding, the request validation — all derived from the JSON or YAML file that describes your API.

Testing such a service should follow the same contract-first philosophy. Instead of manually constructing HTTP requests, hardcoding URLs, and writing ad-hoc response checks, your tests should be driven by the same specification that drives the server. When a path parameter is added, a schema field changes, or a new operation appears, both the server and the tests pick it up automatically.

This post walks you through testing an Apache Camel OpenAPI REST service in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus loads the OpenAPI specification at runtime, sends requests by operation ID instead of URL, auto-generates valid request payloads from schema definitions, and validates responses against the contract — all without writing a single line of JSON or constructing a single URL in the test code.

By the end you will have a recipe for specification-driven API testing that stays in sync with your API contract by design.

# The application under test

The example application implements a Petstore REST API — the classic OpenAPI demonstration service. Instead of defining each endpoint manually, it uses Camel's contract-first `rest().openApi()` DSL to generate all routes from a `petstore-api.json` specification file.

The entire application fits in a single route builder:

```java
public class PetstoreRoutes extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        restConfiguration()
                .clientRequestValidation(true)
                .apiContextPath("openapi");

        rest()
            .openApi()
            .specification("petstore-api.json")
            .missingOperation("mock");
    }
}
```

There is remarkably little code here, but each line does something important.

The `restConfiguration()` block sets up two things. The `clientRequestValidation(true)` setting tells Camel to validate every incoming request against the OpenAPI schema before it reaches any route logic. If a client sends a request with a missing required field or an invalid data type, Camel rejects it with a 400 Bad Request — no custom validation code needed. The `apiContextPath("openapi")` setting exposes the full OpenAPI specification at the `/openapi` endpoint, making the contract available for runtime discovery.

The `rest().openApi()` block is where the magic happens. The `.specification("petstore-api.json")` call points Camel at the OpenAPI file on the classpath, and Camel generates REST endpoints for every operation defined in the spec. The `.missingOperation("mock")` setting tells Camel what to do when an operation has no explicit route implementation: instead of failing, it returns mock responses loaded from classpath resources.

The OpenAPI specification defines four operations on the Pet resource:

| Operation | Method | Path | Operation ID |
|-----------|--------|------|--------------|
| Find a pet | GET | `/petstore/pet/{petId}` | `getPetById` |
| Add a pet | POST | `/petstore/pet` | `addPet` |
| Update a pet | PUT | `/petstore/pet` | `updatePet` |
| Delete a pet | DELETE | `/petstore/pet/{petId}` | `deletePet` |

For the `getPetById` operation, the mock mode loads a response from `examples/pet/1000.json` on the classpath:

```json
{
  "id": 1000,
  "name": "fluffy",
  "category": {
    "id": 1000,
    "name": "dog"
  },
  "photoUrls": [
    "petstore/v3/photos/1000"
  ],
  "tags": [
    {
      "id": 1000,
      "name": "generated"
    }
  ],
  "status": "available"
}
```

This mock pattern is configured via a single application property:

```properties
camel.component.rest-openapi.mock-include-pattern=classpath:examples/**
```

The result is a fully functional REST API with request validation, schema-compliant responses, and a live OpenAPI endpoint — all generated from the specification with zero hand-written endpoint code.

# Adding Citrus to the project

This example uses Citrus's OpenAPI module, which is specifically designed for specification-driven testing:

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
    <artifactId>citrus-openapi</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-validation-json</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Here is what each module brings to the table:

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle.
- **citrus-http** provides the HTTP client that sends requests and receives responses.
- **citrus-openapi** is the key module — it provides the `openapi()` DSL that lets you reference operations by ID, auto-generate request payloads from schemas, and validate responses against the specification.
- **citrus-validation-json** handles JSON response body validation.

The **citrus-openapi** module is what sets this example apart from the other samples on this site. Instead of building requests manually with `http().client().send().post("/petstore/pet")`, you simply say `openapi().send("addPet")` and Citrus figures out the method, the path, the content type, and the request body from the specification.

# Setting up the test class

The test class is minimal — no endpoint declarations, no container annotations, no lifecycle listeners:

```java
@QuarkusTest
@CitrusSupport
class PetstoreOpenApiTest {

    @CitrusResource
    TestCaseRunner t;

    OpenApiSpecification petstoreApi =
            OpenApiSpecification.from("http://localhost:8081/openapi");
}
```

The `OpenApiSpecification` is loaded from the running application's `/openapi` endpoint. This is a deliberate design choice: the test fetches the specification from the same source that serves it to real API consumers. If the specification changes — a new field becomes required, an operation is renamed, a path parameter is added — the test automatically picks up the change because it reads the live contract.

The `TestCaseRunner` (named `t` for brevity) provides the Given-When-Then structure for test actions. Unlike the other Camel samples that use `GherkinTestActionRunner`, this test uses the standard runner, which works the same way with a slightly different naming style.

There are no `@CitrusEndpoint` annotations in this test. The OpenAPI DSL handles endpoint creation internally — you just point it at the server URL and the specification, and Citrus manages the HTTP client.

# Testing GET — find a pet by ID

The first test retrieves a pet and validates the response against the OpenAPI schema:

```java
@Test
void shouldGetPetById() {
    t.given(
        createVariables()
            .variable("petId", "1000")
    );

    t.when(
        openapi().specification(petstoreApi)
                .client("http://localhost:8081")
                .send("getPetById")
    );

    t.then(
        openapi().specification(petstoreApi)
                .client("http://localhost:8081")
                .receive("getPetById", HttpStatus.OK)
    );
}
```

Three steps, each doing a lot behind the scenes.

The **given** block creates a test variable named `petId` with value `"1000"`. This variable is not just a local value — Citrus uses it to resolve the `{petId}` path parameter defined in the OpenAPI specification. When the specification says the `getPetById` operation has a path `/pet/{petId}`, Citrus looks for a test variable with that name and substitutes it automatically.

The **when** block sends the request. The `openapi().send("getPetById")` call tells Citrus to look up the `getPetById` operation in the specification, determine that it is a GET request to `/petstore/pet/{petId}`, resolve the path parameter from the test variable, and send the request to `http://localhost:8081/petstore/pet/1000`. You never write the URL, the HTTP method, or the content type — the specification provides all of it.

The **then** block receives and validates the response. The `openapi().receive("getPetById", HttpStatus.OK)` call tells Citrus to expect a 200 OK response and validate the response body against the Pet schema defined in the OpenAPI specification. Citrus checks that all required fields (`name`, `category`, `status`) are present, that field types match (integers are integers, strings are strings, arrays are arrays), and that enum values fall within the allowed set. If the response body contains `"status": "unknown"` instead of `"available"`, `"pending"`, or `"sold"`, the validation fails automatically.

# Testing POST — add a new pet

Creating a resource requires a request body, but Citrus generates one from the schema:

```java
@Test
void shouldAddPet() {
    t.when(
        openapi().specification(petstoreApi)
                .client("http://localhost:8081")
                .send("addPet")
    );

    t.then(
        openapi().specification(petstoreApi)
                .client("http://localhost:8081")
                .receive("addPet", HttpStatus.OK)
    );
}
```

There is no request body in the test code — and that is the point. The `addPet` operation in the OpenAPI specification defines a required request body with a Pet schema that includes required fields (`name`, `category`, `status`), optional fields (`id`, `photoUrls`, `tags`), and specific types and constraints for each.

When Citrus encounters `send("addPet")`, it reads the schema, generates a valid JSON object that satisfies all the constraints, and sends it as the POST body. The generated payload includes proper values for every required field, correct data types, and valid enum values. You get a complete, schema-compliant request without writing any JSON.

This auto-generation has a practical benefit beyond convenience: it tests the server's ability to accept a schema-compliant request. If the server rejects a valid request, the test fails — catching bugs in the server's request handling that manual tests with hand-crafted payloads might miss.

# Testing PUT and DELETE

The update and delete operations follow the same pattern:

```java
@Test
void shouldUpdatePet() {
    t.when(
        openapi().specification(petstoreApi)
                .client("http://localhost:8081")
                .send("updatePet")
    );

    t.then(
        openapi().specification(petstoreApi)
                .client("http://localhost:8081")
                .receive("updatePet", HttpStatus.OK)
    );
}

@Test
void shouldDeletePet() {
    t.given(
        createVariables()
            .variable("petId", "1000")
    );

    t.when(
        openapi().specification(petstoreApi)
                .client("http://localhost:8081")
                .send("deletePet")
    );

    t.then(
        openapi().specification(petstoreApi)
                .client("http://localhost:8081")
                .receive("deletePet", HttpStatus.OK)
    );
}
```

The `updatePet` test is identical in structure to `addPet` — Citrus generates a valid Pet JSON body and sends it as a PUT request. The `deletePet` test mirrors `getPetById` — it sets a `petId` variable that Citrus resolves into the path parameter, then sends a DELETE request to `/petstore/pet/1000`.

Every test follows the same three-step structure: optionally set variables, send a request by operation ID, and receive a response with schema validation. The consistency is not accidental — it is a direct consequence of the specification-driven approach. Since the specification describes everything about each operation, the tests only need to name the operation and declare the expected outcome.

# What the OpenAPI DSL does for you

It is worth stepping back to appreciate how much work the `openapi()` DSL handles automatically. In a traditional HTTP test, each of these concerns is your responsibility:

**Request construction** — the DSL reads the operation's HTTP method, path, content type, and parameter definitions from the specification. You never write `post("/petstore/pet")` or set a `Content-Type: application/json` header.

**Path parameter resolution** — when a path contains `{petId}`, the DSL looks for a Citrus test variable with that name and substitutes it. No string concatenation, no URL building.

**Request body generation** — for operations that require a body, the DSL generates a valid JSON object from the schema definition. Required fields are populated, data types are correct, and enum values are valid.

**Response schema validation** — the DSL validates the response body against the schema defined for the expected status code. Field names, types, required fields, and enum constraints are all checked.

**Contract synchronization** — since the specification is loaded from the running application at `http://localhost:8081/openapi`, any change to the contract is immediately reflected in the tests. There is no stale copy to keep in sync.

This means that when you add a new required field to the Pet schema, the server's request validation starts enforcing it, the mock responses need to include it, and the tests automatically generate it in requests and validate it in responses — all from a single change in the specification file.

# How it all fits together

When you run the tests with `./mvnw clean test`, here is the sequence of events:

1. **Quarkus starts** the application in test mode on port 8081.
2. **Apache Camel reads** `petstore-api.json` and generates REST endpoints for all four operations.
3. **The OpenAPI spec** is served at `http://localhost:8081/openapi` for runtime discovery.
4. **Citrus loads** the specification from the application's `/openapi` endpoint.
5. **Each test** references an operation by ID (e.g., `getPetById`), and Citrus resolves the method, path, parameters, and body from the spec.
6. **The request** is sent to the Camel REST endpoint, which validates it against the schema.
7. **Camel's mock handler** returns a response (from classpath resources or a generated default).
8. **Citrus validates** the response against the schema defined for the expected status code.

The entire test suite runs in under three seconds with no containers, no external services, and no hand-written JSON. Four operations — GET, POST, PUT, DELETE — are tested with full schema validation in about 30 lines of test code.

There is one piece of test configuration worth mentioning:

```properties
quarkus.arc.ignored-split-packages=org.citrusframework.*
```

This avoids CDI bean discovery conflicts between Citrus and Quarkus.

# Contract-first vs code-first testing

This specification-driven approach inverts the usual testing workflow. In code-first testing, you write the server, then write tests that exercise specific URLs with specific payloads. If the API changes, you update both. In contract-first testing, the specification is the single source of truth — the server and the tests are both derived from it.

The practical difference shows up in maintenance. When you add a new operation to the OpenAPI specification:

- **Server side**: Camel generates the endpoint automatically (or you add a route for it).
- **Test side**: you write one test method with `send("newOperationId")` and `receive("newOperationId", HttpStatus.OK)`.

When you modify an existing schema:

- **Server side**: request validation and mock responses update automatically.
- **Test side**: request generation and response validation update automatically.

The specification change propagates through both sides without manual synchronization. This is especially valuable in larger teams where the API contract is defined by one team and consumed by several others — everyone works against the same source of truth.

# Where to go from here

This example covers the happy path with mock responses. Here are several directions you can take it:

- **Custom request bodies**: Override the auto-generated body with a specific payload using `.message().body(...)` on the send action, while still letting Citrus handle the URL and method resolution.
- **Error status codes**: Test 400 (invalid request), 404 (not found), and 405 (method not allowed) responses by sending requests that violate schema constraints — the server's `clientRequestValidation(true)` setting rejects them automatically.
- **Real route implementations**: Replace the `missingOperation("mock")` setting with actual Camel route implementations that handle business logic, and test the complete processing pipeline.
- **Multiple specifications**: If your application serves multiple APIs, create separate `OpenApiSpecification` instances for each and test them independently in the same test class.
- **External HTTP services**: If your Camel routes call external APIs as part of handling a request, combine this approach with the [HTTP service virtualization sample](/samples/camel-quarkus-direct-http/) to simulate those dependencies while testing the contract-first endpoints.
- **Simpler routes**: If your Camel routes use direct endpoints without REST or OpenAPI, the [direct endpoint sample](/samples/camel-quarkus-direct/) demonstrates the simplest possible testing setup.

To explore the full example project including the source code and the OpenAPI specification, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers OpenAPI integration, HTTP endpoints, JSON validation, and the many other transports that Citrus supports.
