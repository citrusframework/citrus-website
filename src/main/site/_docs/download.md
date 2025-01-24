---
layout: docs
title: Download
permalink: /docs/download/
---

Citrus ${citrus.version} is the latest stable release . Citrus is available on [Maven central repository](http://search.maven.org/#search%7Cga%7C1%7Corg.citrusframework) 
so you can add Citrus as a [Maven dependency](#maven) to your project.

## Maven 

You can easily use Citrus in a Maven project by adding the Citrus modules as test-scoped dependencies in your `pom.xml`.

Citrus provides a Maven BOM or "Bill Of Materials" that controls the versions and available modules that you can add to your project.

{% highlight xml %}
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-bom</artifactId>
  <version>${citrus.version}</version>
  <scope>import</scope>
</dependency>
{% endhighlight %}

Now you can add individual Citrus modules to your project. Each module represents a specific features and capability in Citrus.

As an example the following modules enable you to send and receive messages via Http REST, Kafka and JMS.

{% highlight xml %}
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-http</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-kafka</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-jms</artifactId>
  <scope>test</scope>
</dependency>
{% endhighlight %}

In case you want to use the Apache Camel or Testcontainers functionality provided in Citrus you may add these modules.

{% highlight xml %}
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-camel</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-testcontainers</artifactId>
  <scope>test</scope>
</dependency>
{% endhighlight %}

You can write Citrus test as YAML files using a domain specific language. To use this feature in Citrus add the following module:

{% highlight xml %}
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-yaml</artifactId>
  <scope>test</scope>
</dependency>
{% endhighlight %}

Validation of received messages is an essential part of the test. Citrus provides a powerful set of different message validators for different message data formats.

As an example the following modules add validation capabilities for Json, Xml and plaintext.

{% highlight xml %}
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-validation-json</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-validation-xml</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.citrusframework</groupId>
  <artifactId>citrus-validation-text</artifactId>
  <scope>test</scope>
</dependency>
{% endhighlight %}

You can choose from a huge list of modules that all add individual Citrus capabilities to your test.

## Release artifacts

All available versions and production releases for manual download are listed below:

| Version | Release date | Sources |
|:--------|:--------|:--------|
{% for release in site.data.releases limit:12 %}| {{ release.version }} | {{ release.date }} | [zip](https://github.com/citrusframework/citrus/archive/refs/tags/v{{ release.version }}.zip)/[tar.gz](https://github.com/citrusframework/citrus/archive/refs/tags/v{{ release.version }}.tar.gz) |
{% endfor %}

Since Citrus 4.0 the project requires Java 17 (or newer version) to run.

## Logging framework notice

We use [SLF4J](http://www.slf4j.org/) as logging abstraction framework, which means that you as a user are not forced to use a specific logging 
implementation. SLF4J is similar to commons-logging, so you may use whatever logging framework you want to. All you have
to do is add an SLF4J logging implementation to your classpath.

In case you are currently using [log4j2](http://logging.apache.org/log4j) as logging framework just include `slf4j-log4j12.jar` on your classpath and Citrus 
will use `log4j2` too. If you want to use some other framework than please see the [SLF4J](http://www.slf4j.org/) documentation for help.
