---
layout: docs
title: Documentation
permalink: /docs/documentation/
redirect_from: /docs/index.html
---

We have online and offline documentation as HTML and PDF format. The user guide offers comprehensive descriptions and 
code examples to all Citrus features on board. In case you miss something in our documentation please tell us. Also 
in case you discover something wrong or unclear please do not hesitate to tell us. Find below the reference documentation 
for the latest Citrus release version.

## Release documentation

| Version | Documentation |
|:--------|:------|
{% for release in site.data.releases limit:12 %}| {{ release.version }} | [HTML](${context.path}/citrus/reference/{% if release.tag != "latest" %}{{ release.version }}/{% endif %}html/index.html) \| [PDF](${context.path}/citrus/reference/{% if release.tag != "latest" %}{{ release.version }}/{% endif %}pdf/citrus-reference{% if release.tag != "latest" %}-{{ release.version }}{% endif %}.pdf) |
{% endfor %}

## Additional documentation material

- [News](${context.path}/news)
- [Java Sources](https://github.com/citrusframework/citrus)
- [Quickstart Maven](${context.path}/docs/setup-maven)
- [Quickstart Gradle](${context.path}/docs/setup-gradle)
- [Quickstart Ant](${context.path}/docs/setup-ant)
- [Blog](https://labs.consol.de/tags/citrus)
- [Release History](${context.path}/docs/history/)


## Contribute changes

In case you would like to checkout the Citrus code base and build Citrus yourself follow these instructions:

- [Development quickstart](${context.path}/docs/development)
- [Coding conventions](${context.path}/docs/conventions)
- [How to contribute](${context.path}/docs/contribute)
