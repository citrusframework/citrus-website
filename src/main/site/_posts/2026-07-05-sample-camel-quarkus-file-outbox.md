---
layout: sample
title: Apache Camel Quarkus - File Aggregation Outbox
name: camel-file-outbox
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel Aggregator EIP with file output in Quarkus using Citrus
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-file-outbox/
---

The Aggregator is one of the most powerful enterprise integration patterns. It collects individual messages over time, combines them according to a strategy you define, and releases a single consolidated result when a completion condition is met. Apache Camel makes this pattern easy to implement — a few lines of route definition and a small strategy class handle the entire lifecycle of collecting, merging, and releasing messages.

But testing aggregation logic requires a different approach than testing simple pass-through routes. You need to send multiple messages, trigger the completion condition, and then verify that the aggregated output matches what you expect. When the output is a file on disk, you also need a way to read and validate file content within the test framework.

This post walks you through testing an Apache Camel aggregation route in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. You will see how Citrus sends multiple messages to trigger aggregation, consumes the output file using Camel's file endpoint, and validates the aggregated content. 

By the end you will have a clear recipe for testing Camel routes that use the Aggregator EIP.

# The application under test

The example application implements a task collection service. It receives individual task messages through a direct endpoint, aggregates them into a multi-line text block, and writes the result to a file once three tasks have been collected.

```
Direct Endpoint (direct:tasks) --> Aggregate (3 messages) --> File (outbox/tasks.txt)
```

The route and the aggregation strategy are defined together in a single class:

```java
public class Routes extends EndpointRouteBuilder {

    @Override
    public void configure() throws Exception {
        from(direct("tasks"))
            .aggregate(constant(true), new MultilineAggregationStrategy())
                .completionSize(3)
                .to(file("outbox").fileName("tasks.txt"));
    }

    static class MultilineAggregationStrategy implements AggregationStrategy {
        public Exchange aggregate(Exchange oldExchange, Exchange newExchange) {
            if (oldExchange == null) {
                return newExchange;
            }

            String oldBody = oldExchange.getIn().getBody(String.class);
            String newBody = newExchange.getIn().getBody(String.class);
            oldExchange.getIn().setBody(oldBody + "\n" + newBody);
            return oldExchange;
        }
    }
}
```

Let's walk through the route step by step.

The route starts with `direct("tasks")`, a synchronous in-memory endpoint that serves as the entry point for task messages. Messages arrive here from within the application or, during testing, from Citrus.

The **Aggregator EIP** is the core of the route. It takes two arguments: a correlation expression and an aggregation strategy. The correlation expression `constant(true)` groups all incoming messages together — every message goes into the same aggregation bucket. In a more complex scenario, you could correlate by a header value or a body field to maintain separate aggregation groups.

The `MultilineAggregationStrategy` defines how messages are merged. When the first message arrives, `oldExchange` is null, so the strategy simply returns the new message as the initial aggregate. For each subsequent message, it concatenates the new body to the existing aggregate with a newline separator. After three messages, the aggregate might look like:

```
Doctor's appointment 9:00am
Fetch kids from school
Plan next vacation in June
```

The `completionSize(3)` condition tells the aggregator to release the consolidated message after collecting exactly three inputs. Once released, the aggregated message flows to `file("outbox").fileName("tasks.txt")`, which writes the content to a file named `tasks.txt` in the `outbox` directory.

Notice that this example has no external dependencies — no Kafka, no JMS, no brokers. It is a pure in-memory-to-file pipeline, which makes it a good starting point for understanding the Aggregator EIP before adding messaging transports.

# Adding Citrus to the project

Since this example only uses Camel's direct and file endpoints, the dependency list is minimal:

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
    <artifactId>citrus-junit-jupiter</artifactId>
    <version>${citrus.version}</version>
    <scope>test</scope>
</dependency>
```

Here is what each module brings to the table:

- **citrus-quarkus** integrates Citrus with the Quarkus test lifecycle.
- **citrus-camel** provides the `camel()` DSL for sending messages to Camel direct endpoints, consuming files through Camel's file endpoint, and using Camel processors for type conversion during verification.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

No Kafka module, no Testcontainers, no broker dependencies. The Citrus-Camel integration handles both the input side (sending to `direct:tasks`) and the output side (reading from the `outbox` directory).

# Setting up the test class

The test class wires together Quarkus, Camel, and Citrus:

```java
@QuarkusTest
@CitrusSupport
class QuarkusApplicationTest extends CamelQuarkusTestSupport
        implements TestActionSupport {

    @Inject
    @BindToRegistry
    CamelContext camelContext;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

The class extends `CamelQuarkusTestSupport`, which provides Camel testing capabilities on top of the Quarkus test lifecycle. The `CamelContext` is injected with `@Inject` and registered in the Citrus registry with `@BindToRegistry`, giving the test access to Camel's processing capabilities — including the type converters needed to read file content as text.

There are no endpoint declarations in the setup. The input side uses Camel's `direct:tasks` endpoint by URI, and the output side uses Camel's file endpoint inline in the test method. This keeps the test class lean when the endpoints are simple.

# Writing the test

The test sends three task messages to trigger aggregation, then verifies the output file:

```java
@Test
void shouldAggregateFileContent() {
    runner.when(
        Arrays.asList(
            send()
                .endpoint("camel:direct:tasks")
                .message()
                .body("Doctor's appointment 9:00am"),
            send()
                .endpoint("camel:direct:tasks")
                .message()
                .body("Fetch kids from school"),
            send()
                .endpoint("camel:direct:tasks")
                .message()
                .body("Plan next vacation in June")
        )
    );

    runner.then(
        camel()
            .receive()
            .endpoint("file:outbox")
            .process(processor().camel()
                    .convertBodyTo(String.class))
            .message()
            .header("CamelFileName", "tasks.txt")
            .body("""
            Doctor's appointment 9:00am
            Fetch kids from school
            Plan next vacation in June
            """)
    );
}
```

Let's break this down into the two phases.

## Sending messages to trigger aggregation

```java
runner.when(
    Arrays.asList(
        send().endpoint("camel:direct:tasks").message()
              .body("Doctor's appointment 9:00am"),
        send().endpoint("camel:direct:tasks").message()
              .body("Fetch kids from school"),
        send().endpoint("camel:direct:tasks").message()
              .body("Plan next vacation in June")
    )
);
```

The `Arrays.asList(...)` wrapper sends three messages in sequence to the `camel:direct:tasks` endpoint. Each `send()` action delivers one task message to the Camel route's direct consumer. The `camel:` prefix tells Citrus to route the message through the shared Camel context.

As the messages arrive, the aggregator collects them:

```
Message 1 arrives → Aggregate = "Doctor's appointment 9:00am"
Message 2 arrives → Aggregate = "Doctor's appointment 9:00am\nFetch kids from school"
Message 3 arrives → Aggregate complete → File written to outbox/tasks.txt
```

After the third message, `completionSize(3)` triggers, and the aggregated content is written to disk.

## Verifying the output file

```java
runner.then(
    camel()
        .receive()
        .endpoint("file:outbox")
        .process(processor().camel()
                .convertBodyTo(String.class))
        .message()
        .header("CamelFileName", "tasks.txt")
        .body("""
        Doctor's appointment 9:00am
        Fetch kids from school
        Plan next vacation in June
        """)
);
```

The verification side is where the Citrus-Camel integration shines. The `camel().receive()` action uses Camel's file endpoint to consume the output file from the `outbox` directory. But raw file content arrives as a byte stream, not as a string. The `process(processor().camel().convertBodyTo(String.class))` step uses Camel's type conversion system to transform the bytes into text before Citrus validates the content.

The assertion checks two things: the file name header matches `"tasks.txt"`, and the body matches the expected multi-line text block. Java's text block syntax (`"""..."""`) makes the expected content readable and easy to maintain — it preserves the exact whitespace and newline structure of the aggregated output.

If any message is missing, out of order, or incorrectly merged, the body comparison fails with a clear diff showing exactly where the actual content diverges from the expectation.

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events:

1. **Quarkus starts** the application in test mode.
2. **Apache Camel discovers** the route and initializes the aggregator with `completionSize(3)`.
3. **Citrus sends** three task messages to `camel:direct:tasks` in sequence.
4. **The aggregator collects** each message, merging them with newline separators.
5. **After the third message**, the completion condition triggers and the aggregated content is released.
6. **The file producer** writes `tasks.txt` to the `outbox` directory.
7. **Citrus consumes** the file from `outbox` using Camel's file endpoint.
8. **Camel's type converter** transforms the file bytes to a string.
9. **Citrus validates** the file name and the multi-line body content.

This example has no external infrastructure — no containers, no brokers, no network calls. The entire test runs in-process, which makes it fast (typically under two seconds) and deterministic.

# The Aggregator and the Splitter — two sides of the same coin

If you have read the [file processing pipeline sample](/samples/camel-quarkus-file-inbox/) on this site, you will notice that this example is its mirror image. The file inbox sample uses the **Splitter EIP** to break a single file into multiple Kafka messages. This file outbox sample uses the **Aggregator EIP** to combine multiple messages into a single file.

From a testing perspective, the patterns are symmetric:

- **Splitter test**: send one message, verify multiple outputs
- **Aggregator test**: send multiple messages, verify one output

Citrus handles both patterns with the same `send()` and `receive()` actions. The only difference is how many of each you need. This symmetry extends to real-world applications — many integration pipelines split incoming data for parallel processing and then re-aggregate results for downstream consumption. Both sides can be tested with the same Citrus approach.

# Where to go from here

This example covers the basics of the Aggregator EIP, but there is much more to explore:

- **Different completion conditions**: Replace `completionSize(3)` with `completionTimeout(5000)` for time-based aggregation, or use `completionPredicate(simple("${body} contains 'DONE'"))` for content-based completion. Each condition changes how you structure the test.
- **Correlation expressions**: Replace `constant(true)` with a header-based expression to maintain separate aggregation groups. Test that messages with different correlation keys produce separate output files.
- **Custom aggregation strategies**: Try collecting messages into a JSON array, computing a sum, or selecting the best message based on a score. The strategy defines the merge logic; the test validates the result.
- **File append mode**: Add `.fileExist("Append")` to the file endpoint to accumulate across multiple aggregation cycles instead of overwriting.
- **Messaging transports**: If your aggregation route consumes from Kafka or JMS instead of a direct endpoint, the [Camel Kafka sample](/samples/camel-quarkus-kafka/) and [Camel JMS sample](/samples/camel-quarkus-jms/) on this site demonstrate how to set up broker infrastructure for testing.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers Camel integration, file endpoints, data format processors, and the many other transports that Citrus supports.
