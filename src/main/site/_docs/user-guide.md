---
layout: docs
title: User Guide
permalink: /docs/user-guide/
---

We have online and offline documentation as HTML and PDF format. The user guide offers comprehensive descriptions and 
code examples to all Citrus features on board. In case you miss something in our documentation please tell us. Also 
in case you discover something wrong or unclear please do not hesitate to tell us. Find below the reference documentation 
for the latest Citrus release version.

## Release documentation

| Version | Documentation |
|:--------|:------|
{% for release in site.data.releases limit:12 %}| {{ release.version }} | [HTML](https://christophd.github.io/citrus/reference/{% if release.tag != "latest" %}{{ release.version }}/{% endif %}html/index.html) \| [PDF](https://christophd.github.io/citrus/reference/{% if release.tag != "latest" %}{{ release.version }}/{% endif %}pdf/citrus-reference{% if release.tag != "latest" %}-{{ release.version }}{% endif %}.pdf) |
{% endfor %}

## Additional documentation material

- [News](${site.path}/news)
- [Java Sources](http://www.github.com/christophd/citrus)
- [Quickstart Maven](${site.path}/docs/setup-maven)
- [Quickstart Gradle](${site.path}/docs/setup-gradle)
- [Quickstart Ant](${site.path}/docs/setup-ant)
- [Blog](https://labs.consol.de/tags/citrus)
- [Release History](${site.path}/docs/history/)


## Contribute changes

In case you would like to checkout the Citrus code base and build Citrus yourself follow these instructions:

- [Development quickstart](${site.path}/docs/development)
- [Coding conventions](${site.path}/docs/conventions)
- [How to contribute](${site.path}/docs/contribute)