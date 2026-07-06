---
layout: sample
title: Apache Camel Quarkus - HTTP to PostgreSQL
name: camel-postgresql
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing a Camel HTTP-to-PostgreSQL route in Quarkus with Citrus HTTP client and SQL validation
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-postgresql/
---

An HTTP endpoint that accepts data and writes it to a database is one of the most common patterns in modern applications. It sounds simple, but testing it properly is not. You need to verify not only the HTTP response but also that the data actually landed in the database with the correct values. A test that checks only the response code tells you half the story.

This post walks through testing an [Apache Camel](https://camel.apache.org) route in a [Quarkus](https://quarkus.io) application that receives JSON over HTTP and persists it into PostgreSQL using a Camel Kamelet. [Citrus](https://citrusframework.org) handles both sides of the verification — the HTTP response and the database state — in a single, self-contained test. The database is provisioned automatically by Quarkus Dev Services, so there is no external infrastructure to manage.

By the end you will have a recipe for testing any data persistence route with full end-to-end confidence.

# The application under test

The application exposes a single HTTP endpoint that accepts headline data and stores it in a PostgreSQL database. The entire flow fits in one Camel route:

```
HTTP POST /headline  -->  Apache Camel Route  -->  PostgreSQL (via postgresql-sink Kamelet)
```

Here is the route definition:

```java
public class Routes extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("platform-http:/headline?httpMethodRestrict=POST")
            .to("kamelet:postgresql-sink?" +
                    "serverName={{jdbc.server.host}}&" +
                    "serverPort={{jdbc.server.port}}&" +
                    "username={{jdbc.username}}&" +
                    "password={{jdbc.password}}&" +
                    "databaseName={{jdbc.database.name}}&" +
                    "query=INSERT INTO headlines " +
                    "VALUES (:#id,:#headline)")
            .setBody().constant("Headline created!")
            .setHeader(Exchange.HTTP_RESPONSE_CODE, constant(201));
    }
}
```

The route starts with `platform-http`, which integrates with the Quarkus HTTP server to expose a POST endpoint at `/headline`. The incoming JSON body is passed to the `postgresql-sink` Kamelet, which handles the database insert. After a successful insert, the route responds with `"Headline created!"` and HTTP status 201.

The `postgresql-sink` Kamelet deserves a closer look. It is a pre-built Camel component from the [Kamelet catalog](https://camel.apache.org/camel-kamelets/latest/) that simplifies database writes. Under the hood it parses the incoming JSON into a Map, creates its own DBCP2 connection pool from the provided connection parameters, and executes the parameterized SQL query. The `:#id` and `:#headline` placeholders are named parameters that bind to the keys in the parsed JSON Map — so a body like `{"id": 42, "headline": "Camel rocks!"}` inserts the values `42` and `Camel rocks!` into the corresponding columns.

The connection parameters use Camel property placeholders, which are resolved from the Quarkus `application.properties`. This keeps the route portable across environments.

The database table is created at application startup by a small initializer:

```java
@ApplicationScoped
public class DatabaseInit {

    @Inject
    DataSource dataSource;

    void onStart(@Observes StartupEvent ev) throws SQLException {
        try (var conn = dataSource.getConnection();
             var stmt = conn.createStatement()) {
            stmt.execute("CREATE TABLE IF NOT EXISTS headlines " +
                "(id INTEGER PRIMARY KEY, " +
                "headline VARCHAR(255))");
        }
    }
}
```

This uses the Quarkus-managed `DataSource` to create the `headlines` table if it does not exist. In test mode, this DataSource connects to the Dev Services PostgreSQL container — so the table is ready before the first test runs.

# Adding Citrus to the project

Testing this application requires two Citrus modules: one for HTTP and one for SQL.

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
    <artifactId>citrus-sql</artifactId>
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

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle and manages Dev Services property sharing.
- **citrus-http** provides the HTTP client endpoint for sending requests to the Camel route.
- **citrus-sql** provides SQL query and validation actions for verifying database state.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

No Camel-specific Citrus module is needed here because the test interacts with the application purely through its HTTP interface — no direct route invocation required. The Camel route is exercised indirectly when the test sends an HTTP POST.

# Setting up the test class

The test class connects Citrus to the Quarkus application and the Dev Services database:

```java
@QuarkusTest
@CitrusSupport(devServicesProperties = "*")
class QuarkusApplicationTest implements TestActionSupport {

    @CitrusEndpoint
    @HttpClientConfig(requestUrl = "http://localhost:8081")
    HttpClient httpClient;

    @Inject
    DataSource dataSource;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

Three things are worth noting here.

First, the `@CitrusSupport(devServicesProperties = "*")` annotation. The `devServicesProperties = "*"` parameter tells Citrus to accept all configuration properties injected by Quarkus Dev Services. This is how Citrus learns about the database connection that Quarkus provisioned — without it, the two frameworks would not share the same configuration context.

Second, the `@Inject DataSource dataSource`. This injects the Quarkus-managed DataSource — the same one that `DatabaseInit` uses to create the table at startup. The DataSource connects to the Dev Services PostgreSQL container, so when the Citrus test queries the database, it reads from the exact same instance that the Camel route writes to. There is no separate test database.

Third, the HTTP client is configured with `@HttpClientConfig(requestUrl = "http://localhost:8081")`. Port 8081 is the default Quarkus test port. This client sends requests to the application just like any external caller would.

# Writing the test

The test follows a natural Given-When-Then flow: set up test data, call the HTTP endpoint, verify the response, and then verify the database.

```java
@Test
void shouldPersistData() {
    runner.given(
        createVariables()
            .variable("id", "citrus:randomNumber(4)")
            .variable("headline", "Camel rocks!")
    );

    runner.when(
        http().client(httpClient)
            .send()
            .post("/headline")
            .message()
            .body("""
                { "id": ${id}, "headline": "${headline}" }
                """)
            .contentType("application/json")
    );

    runner.then(
        http().client(httpClient)
            .receive()
            .response(HttpStatus.CREATED)
            .message()
            .body("Headline created!")
    );

    runner.then(
        sql().dataSource(dataSource)
            .query()
            .statement(
                "SELECT headline FROM headlines WHERE id=${id}")
            .validate("HEADLINE", "${headline}")
    );
}
```

Let's walk through each step.

## Setting up test variables

```java
runner.given(
    createVariables()
        .variable("id", "citrus:randomNumber(4)")
        .variable("headline", "Camel rocks!")
);
```

The test creates two variables: a random 4-digit `id` using the built-in `citrus:randomNumber()` function, and a fixed `headline` string. These variables are referenced throughout the test with `${id}` and `${headline}`, ensuring the same values are used in the HTTP request, the expected response, and the SQL validation. The random ID prevents collisions between test runs without manual cleanup.

## Sending the HTTP request

```java
runner.when(
    http().client(httpClient)
        .send()
        .post("/headline")
        .message()
        .body("""
            { "id": ${id}, "headline": "${headline}" }
            """)
        .contentType("application/json")
);
```

The test sends a POST request to `/headline` with a JSON body containing the test variables. This triggers the Camel route, which parses the JSON and passes it to the `postgresql-sink` Kamelet for database insertion.

## Verifying the HTTP response

```java
runner.then(
    http().client(httpClient)
        .receive()
        .response(HttpStatus.CREATED)
        .message()
        .body("Headline created!")
);
```

After the route processes the request, the test verifies the HTTP response: status 201 Created and the body `"Headline created!"`. This confirms the route completed successfully — but it does not tell you whether the data actually reached the database. A route that swallows an insert error and still returns 201 would pass this check. That is why the next step is essential.

## Verifying the database state

```java
runner.then(
    sql().dataSource(dataSource)
        .query()
        .statement(
            "SELECT headline FROM headlines WHERE id=${id}")
        .validate("HEADLINE", "${headline}")
);
```

This is where Citrus closes the loop. The `sql()` action executes a SELECT query against the shared DataSource, looking for the row that the Camel route just inserted. The `.validate("HEADLINE", "${headline}")` assertion checks that the `headline` column contains the expected value.

If the Kamelet failed to insert the row, if the JSON parsing lost a field, or if the named parameter binding mapped the wrong key — this step catches it. The test validates the actual side effect, not just the HTTP facade.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events:

1. **Quarkus Dev Services** starts a PostgreSQL container on port 15432.
2. **`@CitrusSupport(devServicesProperties = "*")`** initializes Citrus with the Dev Services configuration.
3. **Quarkus starts** the application in test mode. The `DatabaseInit` bean creates the `headlines` table.
4. **Apache Camel discovers** the route and registers the `platform-http:/headline` endpoint.
5. **The test creates** a random `id` and a `headline` variable.
6. **Citrus sends** an HTTP POST with the JSON body to `/headline`.
7. **The Camel route** receives the request, passes it to the `postgresql-sink` Kamelet, which inserts a row into PostgreSQL.
8. **The route responds** with 201 Created and `"Headline created!"`.
9. **Citrus validates** the HTTP response status and body.
10. **Citrus queries** the database using the shared DataSource and verifies the inserted row.
11. **The test completes.** The PostgreSQL container is automatically stopped.

The key insight is that three components — the Quarkus `DatabaseInit` bean, the Camel Kamelet, and the Citrus SQL action — all connect to the same PostgreSQL instance without any manual wiring. Quarkus Dev Services provisions the database, and the shared configuration makes it available to everyone.

# Dev Services configuration

The test configuration in `src/test/resources/application.properties` pins the Dev Services port so the Kamelet's connection parameters can be statically configured:

```properties
quarkus.datasource.devservices.port=15432

jdbc.server.host=localhost
jdbc.server.port=15432
jdbc.username=quarkus
jdbc.password=quarkus
jdbc.database.name=quarkus
```

The `quarkus.datasource.devservices.port=15432` setting fixes the PostgreSQL container to a known port. The `jdbc.*` properties override the production defaults to match the Dev Services container credentials (username `quarkus`, password `quarkus`, database `quarkus`). This way the Kamelet's DBCP2 connection pool connects to the same database that the Quarkus DataSource manages.

In production, the `jdbc.*` properties point to the real database:

```properties
jdbc.server.host=localhost
jdbc.server.port=5432
jdbc.username=postgres
jdbc.password=postgres
jdbc.database.name=postgres
```

The route itself stays identical across environments — only the property values change.

# Where to go from here

This example tests a single insert operation. Here are directions you can extend it:

- **Multiple inserts**: Send several headlines and use a SQL query with `ORDER BY` to validate all rows were persisted in the correct order.
- **Update and delete**: Add PUT and DELETE endpoints to the route and use `sql().execute()` for setup data and `sql().query()` for validation.
- **Error scenarios**: Send malformed JSON or duplicate IDs and verify that the route returns the appropriate HTTP error codes.
- **Messaging integration**: Replace the HTTP trigger with a Kafka or JMS consumer and combine messaging validation with SQL assertions — the [Kafka sample](/samples/camel-quarkus-kafka/) shows how to set up Kafka endpoints with Citrus.
- **Protocol bridges**: If your route exposes REST and delegates to a SOAP backend before writing to the database, the [REST-to-SOAP sample](/samples/camel-quarkus-rest-to-soap/) demonstrates dual-role testing with HTTP clients and SOAP servers.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers HTTP endpoints, SQL validation, Camel integration, and the many other transports that Citrus supports.
