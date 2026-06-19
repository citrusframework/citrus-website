---
layout: post
title: Get ready for Citrus 5.0.0
short-title: Citrus 5.0.0 approaching
author: Christoph Deppisch
github: christophd
categories: [blog]
---

Citrus 5.0.0 is on the horizon! The next major release of the Citrus integration testing framework is planned for **GA in Q3 2026** and brings a sweeping update of the core technology stack that Citrus builds on. Two milestone releases (M1 and M2) are already available for early adopters who want to get a head start on migration.

## Why a new major version?

The Java ecosystem has taken significant leaps forward in recent time. Spring Framework 7, Spring Boot 4, Kafka 4, JUnit 6 and Jackson 3 all represent new major versions that come with their own set of breaking changes and modern API improvements. Citrus 5 aligns with these releases so you can use the latest and greatest enterprise Java libraries alongside your integration tests.

Here are the main objectives for Citrus 5.0:

* While still supporting Java 17 Citrus aligns with **Java 21** (LTS baseline) and **Java 25** support
* Update to **Spring Framework 7** and **Spring Boot 4**
* Update to **JUnit 6**
* Update to **Jackson 3**, **Apache Camel 4.21**, **Kafka 4.2**, **Jetty 12**
* Introduce a **Citrus MCP server** for AI-assisted test development
* Expand **Quarkus** support with easy to use integration into the Quarkus test lifecycle
* Streamline YAML DSL schema and improve the UX with visual designers such as Kaoto

## Java 17 and Java 21

Citrus 5.0 still supports Java 17 as a minimum Java version but all cursors are pointing towards **Java 21**. This lets the framework and its users take full advantage of modern Java features such as virtual threads, pattern matching, sequenced collections and record patterns.

## Dependency updates

Citrus depends on many outstanding open source libraries and the 5.0 release aligns with their latest major versions. Here is an overview of the most important updates:

| Library               | Version | Library      | Version |
|:----------------------|:--------|:-------------|:--------|
| Spring Framework      | 7.0.7   | Spring Boot  | 4.0.6   |
| Spring Integration    | 7.0.4   | Spring WS    | 5.0.1   |
| Apache Camel          | 4.21.0  | Apache Kafka | 4.2.0   |
| Jackson               | 3.1.0   | Jetty        | 12.1.8  |
| JUnit                 | 6       | Quarkus      | 3.36.2  |

### Spring Framework 7 and Spring Boot 4

Spring 7 brings a refined programming model, improved observability and continued evolution of the Spring module system. Spring Boot 4 builds on top of this foundation with updated auto-configuration and new starter patterns. Citrus 5 is fully aligned with these releases, so you can use Spring Boot 4 in your test projects without dependency conflicts.

### JUnit 6

JUnit 6 is the next generation of the most popular Java testing framework. Citrus 5 ships with a dedicated runtime module for JUnit 6, so you can write your Citrus test cases using the new JUnit 6 APIs and extensions right away.

### Jackson 3

Jackson 3.1.0 represents a major step in the Jackson JSON library with a reworked module system and updated API. Citrus uses Jackson extensively for JSON message validation and data binding, and version 5.0 ensures full compatibility with the new Jackson 3 APIs.

### Apache Kafka 4.2

Kafka 4.2 introduces the new KRaft-based architecture as the default, removing the ZooKeeper dependency entirely. Citrus 5 aligns with this release and also brings new Testcontainers support for Kafka into Citrus.

### Quarkus 3.36

Citrus is fully integrated into the Quarkus test ecosystem with special Quarkus test resource implementations that are able to take part of the Quarkus test lifecycle.

Citrus 5 brings continued enhancements to streamline the Citrus integration into Quarkus for instance by populating **Quarkus dev services properties as test variables** for seamless integration with Quarkus dev services. As an example your Citrus tests might share the Kafka broker configuration from dev services out of the box.

## Citrus MCP server

One of the most exciting additions in Citrus 5 is the new **Model Context Protocol (MCP) server**. MCP is an emerging standard that allows AI coding assistants to interact with development tools in a structured way.

The Citrus MCP server exposes the framework's test vocabulary, endpoint configurations and test action catalog to AI assistants. This enables intelligent code completion, test generation and guided test authoring directly from your IDE or command line.

With M2 the MCP server is also published as a **Claude Code plugin** via JBang, making it easy to integrate Citrus-aware AI assistance into your development workflow.

## Expanded Testcontainers support

Citrus has long embraced Testcontainers for managing external service dependencies in integration tests. Version 5.0 significantly expands this support with new container implementations:

* **Native Apache Kafka container** - a dedicated container implementation as an alternative to the existing Testcontainers Kafka module
* **Strimzi Kafka container** - run Kafka via the Strimzi container image, ideal for teams already using Strimzi in their Kubernetes environments
* **Floci container support** - adds Floci containers to the Citrus Testcontainers module as an alternative AWS test service provider.

## Try it now

The two milestone releases M1 and M2 are available on Maven Central. Add Citrus 5.0.0-M2 to your project to try it out:

```xml
<dependency>
    <groupId>org.citrusframework</groupId>
    <artifactId>citrus-bom</artifactId>
    <version>5.0.0-M2</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

Milestone releases are intended for early adopters to validate their existing test suites against the new dependency stack and to provide feedback before the GA release. We encourage you to try upgrading now so you have plenty of time to adapt your tests before the final release in Q3 2026.

## Provide feedback

We would love to hear from you! If you run into issues during migration or have ideas for improvements, please open an issue on [GitHub](https://github.com/citrusframework/citrus/issues) or start a discussion. Your feedback is invaluable in shaping the GA release.

The Citrus community has once again shown its strength in bringing this major update together. After 18+ years of development Citrus continues to evolve with the Java ecosystem, and we are excited to deliver a framework that stays current with the technologies you use every day. Thank you to everyone who contributed!
