---
layout: sample
title: Apache Camel Quarkus - File Processing Pipeline
name: camel-file-inbox
image: /img/icons/camel.png
folder: apache-camel
group: quarkus
description: Testing Apache Camel file processing with ZIP data formats and Splitter EIP in Quarkus using Citrus
categories: [samples]
repository: citrus-quarkus-examples
permalink: /samples/camel-quarkus-file-inbox/
---

File-based integration is one of the oldest patterns in enterprise software, and it is still everywhere. A process drops a file into a directory, another process picks it up, transforms the content, and publishes the results downstream. Apache Camel has supported this pattern since day one with its file component, data format processors, and enterprise integration patterns like the Splitter.

Testing file processing pipelines is tricky. You need to create files in the right format, drop them into the right directory, wait for the application to pick them up, and then verify the output on the other end. And if the file is compressed — a ZIP archive, for instance — you also need to create valid compressed test data.

This post walks you through testing an Apache Camel file processing route in a Quarkus application using the [Citrus](https://citrusframework.org) integration testing framework. The key insight is that Citrus can leverage Camel's own data format processors to create test data. 

Instead of writing boilerplate code to build ZIP files manually, you let Camel compress the content the same way the production route decompresses it. 

By the end you will have a clear recipe for testing file-based Camel routes with complex data transformations.

# The application under test

The example application implements a file processing pipeline. It polls a directory for ZIP files, decompresses them, splits the content into individual words, transforms each word to uppercase, and publishes the results to a Kafka topic.

```
File Inbox (ZIP) --> Unmarshal --> Split --> Transform --> Kafka Topic (words-out)
```

The entire logic lives in a single Camel route using the type-safe Endpoint DSL:

```java
public class Routes extends EndpointRouteBuilder {

    @Override
    public void configure() throws Exception {
        from(file("inbox").delete(true))
            .unmarshal(dataFormat().zipFile().end())
            .convertBodyTo(String.class)
            .split(body().tokenize(" "))
                .setBody(exchange -> ">> " + exchange.getIn().getBody().toString().toUpperCase())
                .to("kafka:words-out");
    }
}
```

There is a lot happening in these few lines, so let's walk through each step.

The route starts with `file("inbox").delete(true)`, which polls the `inbox` directory for new files and deletes each file after successful processing. By extending `EndpointRouteBuilder`, the route uses Camel's Endpoint DSL — `file("inbox")` is a type-safe method call with IDE auto-completion, not a string URI like `"file:inbox?delete=true"` that could contain typos.

Next, `unmarshal(dataFormat().zipFile().end())` decompresses the ZIP archive using Camel's zipFile data format. Data formats are Camel's abstraction for marshalling (object to format) and unmarshalling (format to object). The zipFile format uses Java's `ZipInputStream` under the hood, but you never deal with that directly.

After converting the decompressed content to a string, the **Splitter EIP** takes over. The expression `body().tokenize(" ")` splits the message body on whitespace, turning a single message containing `"Hello World"` into two separate messages: `"Hello"` and `"World"`. Each token becomes an individual Camel exchange that flows through the rest of the route independently.

Finally, each token is transformed to uppercase with a `">> "` prefix and published to the `words-out` Kafka topic as a separate message.

# Adding Citrus to the project

This example uses the Citrus-Camel integration to send files and Kafka endpoints to verify the output. Add these test-scoped dependencies to the Maven POM:

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
    <artifactId>citrus-kafka</artifactId>
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
- **citrus-camel** provides the `camel()` DSL for sending messages through Camel endpoints and — critically — access to Camel's data format processors for creating test data.
- **citrus-kafka** provides Kafka endpoint implementations for receiving and validating the output messages.
- **citrus-junit-jupiter** connects Citrus to JUnit 5.

The key module is **citrus-camel**. It does not just let you send messages through Camel endpoints — it also gives your tests access to Camel's entire library of data format processors. This is how the test creates ZIP files without any manual compression code.

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

    @CitrusEndpoint
    @KafkaEndpointConfig(topic = "words-out",
            server = "${kafka.bootstrap.servers}")
    KafkaEndpoint wordsOut;

    @CitrusResource
    GherkinTestActionRunner runner;
}
```

A few things are worth noting here.

The class extends `CamelQuarkusTestSupport`, which provides additional Camel testing capabilities on top of the standard Quarkus test lifecycle. The `CamelContext` is injected with `@Inject` and registered in the Citrus registry with `@BindToRegistry`. This shared context is what enables the test to use Camel's data format processors — the test accesses the same Camel runtime that the application uses.

The Kafka endpoint uses `${kafka.bootstrap.servers}` as the server address. This property is provided by Quarkus Kafka dev services, which automatically starts a Kafka container via Testcontainers. Unlike the other Camel samples on this site, there is no explicit `@KafkaContainerSupport` annotation — Quarkus handles the broker provisioning entirely on its own.

# Creating test data with Camel data formats

This is the heart of the example. The test needs to create a valid ZIP file and drop it into the `inbox` directory. Without Camel, you would write something like this:

```java
ByteArrayOutputStream baos = new ByteArrayOutputStream();
ZipOutputStream zos = new ZipOutputStream(baos);
zos.putNextEntry(new ZipEntry("content.txt"));
zos.write("Hello World".getBytes());
zos.closeEntry();
zos.close();
byte[] zipContent = baos.toByteArray();
```

That is six lines of low-level Java I/O just to create a simple ZIP file. With Citrus and Camel, you replace all of it with a single declarative transform:

```java
runner.when(
    camel()
        .send()
        .endpoint("file:inbox")
        .message()
        .header("CamelFileName", "words.zip")
        .body("Hello World")
        .transform(processor().camel()
                .marshal()
                .zipFile())
);
```

Let's break this down.

The `camel().send()` DSL sends a message through a Camel endpoint. The endpoint is the `inbox` file directory — the same directory the application route polls. The `CamelFileName` header tells Camel what to name the file.

The body is just the plain text `"Hello World"`. The transform step is where the magic happens: `processor().camel().marshal().zipFile()` tells Citrus to run the message body through Camel's zipFile data format processor before writing it to disk. Camel compresses the text into a valid ZIP archive, and the resulting `words.zip` file lands in the `inbox` directory ready for the application route to pick up.

This approach has a deep advantage beyond saving lines of code. The test uses the exact same data format implementation as the production route — just in the opposite direction. The route calls `unmarshal()` to decompress; the test calls `marshal()` to compress. If the data format configuration ever changes, both sides stay in sync automatically.

# Writing the test

With the test data creation covered, the full test verifies the entire pipeline from ZIP file to Kafka messages:

```java
@Test
void shouldConsumeFileContent() {
    runner.when(
        camel()
            .send()
            .endpoint("file:inbox")
            .message()
            .header("CamelFileName", "words.zip")
            .body("Hello World")
            .transform(processor().camel()
                    .marshal()
                    .zipFile())
    );

    runner.then(
        receive()
            .endpoint(wordsOut)
            .message()
            .body(">> HELLO")
    );

    runner.then(
        receive()
            .endpoint(wordsOut)
            .message()
            .body(">> WORLD")
    );
}
```

The **when** block creates the ZIP file and drops it into the `inbox` directory. The Camel file consumer detects the new file, decompresses it, splits `"Hello World"` into two tokens, transforms each to uppercase with the `">> "` prefix, and publishes them to Kafka.

The two **then** blocks verify that both messages arrive on the `words-out` Kafka topic in the correct order: first `">> HELLO"`, then `">> WORLD"`. Each `receive()` action blocks until a message arrives or a timeout occurs, ensuring deterministic verification of the Splitter EIP output.

This single test covers the entire processing pipeline:
- File detection and consumption from the `inbox` directory
- ZIP decompression using the zipFile data format
- Content splitting with the Splitter EIP
- Uppercase transformation with prefix formatting
- Kafka publication of each split message
- Message ordering across the split

# How it all fits together

When you run the test with `./mvnw clean test`, here is the sequence of events:

1. **Quarkus starts** the application in test mode.
2. **Kafka dev services** start a Kafka container and expose the bootstrap servers property.
3. **Apache Camel discovers** the route and starts polling the `inbox` directory.
4. **Citrus creates** a ZIP file using Camel's zipFile data format and writes it to `inbox/words.zip`.
5. **The Camel file consumer** detects the new file and begins processing.
6. **The route decompresses** the ZIP archive, converts to string, and splits on whitespace.
7. **Each token** is transformed to uppercase with the `">> "` prefix.
8. **Two Kafka messages** are published to the `words-out` topic: `">> HELLO"` and `">> WORLD"`.
9. **Citrus receives** both messages and validates the bodies.
10. **The file is deleted** from the inbox (because `delete(true)` is set on the file endpoint).
11. **The Kafka container** is stopped and cleaned up automatically.

One piece of test configuration is needed for Quarkus CDI compatibility:

```properties
quarkus.arc.ignored-split-packages=org.citrusframework.*
```

# Beyond ZIP: using other data formats in tests

The pattern shown here works with any Camel data format. If your route unmarshals JSON, you can marshal JSON in the test. If your route decodes Base64, you can encode Base64 in the test. The syntax stays the same — only the data format name changes:

```java
// JSON marshalling
.transform(processor().camel(camelContext).marshal().json())

// Base64 encoding
.transform(processor().camel(camelContext).marshal().base64())

// GZIP compression
.transform(processor().camel(camelContext).marshal().gzip())

// CSV generation
.transform(processor().camel(camelContext).marshal().csv())
```

This consistency means you can test any Camel route that uses data formats without writing custom serialization code. The production route and the test always use the same underlying implementation, which eliminates an entire class of test-vs-production divergence bugs.

# Where to go from here

This example covers file processing with ZIP compression and the Splitter EIP, but Citrus and Camel support many more file-based scenarios:

- **File filters and patterns**: Configure the file consumer with `include("*.zip")` to process only certain file types. Test that non-matching files are correctly ignored.
- **Error handling**: Test what happens when the route encounters an invalid ZIP file or a file with unexpected content. Camel's error handling and dead-letter channels can be verified with Citrus.
- **Multiple files**: Send several ZIP files in sequence and verify that all messages arrive on Kafka in the expected order.
- **Aggregator EIP**: The opposite of the Splitter — combine multiple messages back into a single file. Test the round-trip by splitting and re-aggregating.
- **Messaging comparison**: If your Camel route uses JMS or Kafka input instead of file polling, the [Camel JMS sample](/samples/camel-quarkus-jms/) and [Camel Kafka sample](/samples/camel-quarkus-kafka/) on this site demonstrate the equivalent setup.

To explore the full example project including the source code, head over to the [citrus-quarkus-examples](https://github.com/citrusframework/citrus-quarkus-examples) repository on GitHub.

For deeper dives into Citrus capabilities, the [reference guide](https://citrusframework.org/citrus/reference/html/) covers Camel integration, Kafka endpoints, data format processors, and the many other transports that Citrus supports.
