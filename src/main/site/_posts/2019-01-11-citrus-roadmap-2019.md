---
layout: post
title: Roadmap for 2019
short-title: Roadmap for 2019
author: Sven Hettwer
github: svettwer
categories: [blog]
---

**Happy new year everybody**

The year has just started and I'm happy that we're finally ready to share the roadmap for Citrus in 2019 with you. 
As promised via [twitter](https://twitter.com/citrus_test/status/1073162157259440128), this year
*[...] will be about community, enhancing documentation and improving the ease of use of Citrus with a focus on the 
API, the tooling and the commonly used modules*. With more than ten years of history, Citrus has grown from a utility
framework in a customers project into a full fledged message based integration testing framework with the capability of
testing complex software systems supporting a vast amount of communication protocols and messaging technologies.
Every year, the feature set and the number of available modules has been grown and because of this, Citrus has become the
powerful, reliable and flexible testing tool it is today. And this year, we want to make Citrus even more awesome!

In this post, I'll go through all the points mentioned and provide some details. The roadmap only contains the major topics 
we want to approach. We'll still answer questions, fix bugs, add enhancements and smaller features. We'll also create
epics on GitHub for every major topic to make the progress transparent to the community. Please notice that the time
ranges marked in the timeline are more a rough estimation to give you an idea when we want to work on a certain topic.  
But before we go into the details of the roadmap, let's have a look at some other things that we're going to change.

## Release schedule
We're planning to release a new Citrus version constantly every two months starting with the release of *v2.7.9* and
*v2.8.0* this January. This will provide a constant stream Ich of improvements and new features to you.
In addition, we're planning to release a major version of the framework once a year if required, to be able to improve
the API based on the feedback of the community and perform major version updates of our dependencies. 

## High-Priority- and Low-Priority-Modules
During the lasts months, we've monitored the download statistics from the maven central repository and recognized that
some modules are significantly more popular than others. From today on, we want to focus on the modules that are heavily
used by the community. Therefore we'll divide the Citurs modules into *High-Priority-Modules* and *Low-Priority-Modules*.
Newer Modules are also High-Priority as they just have to establish first. We'll invest the major part of our time into the 
High-Priority-Modules to improve the documentation, samples and GitHub support and also focus on bug fixes and new features.
The Low-Priority-Modules on the other hand, will take a back seat concerning this topics. If you're interested to help
us maintaining one of the modules, feel free to contact us and propose pull requests. We always appreciate any kind of
contribution.    

### High-Priority-Modules
* Camel
* Cucumber
* FTP
* HTTP
* JDBC
* JMS
* Kafka
* Mail
* Selenium
* SSH
* Vert.x
* Websockets
* SOAP

### Low-Priority-Modules
* Arquillian
* Docker
* JXM
* Kubernetes
* Restdocs
* RMI
* Zookeeper

## Roadmap 2019

[ ![roadmap-2019.png](${context.path}/img/citrus/roadmap-2019.png)](${context.path}/img/citrus/roadmap-2019.png){:target="_blank"}

### Citrus-JDBC Improvements
Based on the feedback we received last year, we're going to improve the Citrus-JDBC functionality. This includes
output parameters in callable statements, JDBC batch statements, various data types and much more.

### Reduce/review Citrus samples
The [citrus-samples](https://github.com/citrusframework/citrus-samples) repository contains a vast amount of samples in
XML and Java representing a good amount of the Citrus functionality. But this also makes it hard for new users to get
orientation. That's why we want to review the Citrus samples completely and create a set that reflects the functionality
of the framework while making them easy to understand and a good point to start from. To archive this, we'll focus on samples
for the High-Priority-Modules and reduce the amount of samples for Low-Priority-Modules.

### Deprecate XML-DSL
In order to improve the maintainability of the framework and reduce the complexity from a users perspective, we're going
to deprecate the XML-DSL. The use case for the XML-DSL was to provide a way to create integration tests without any knowledge
of Java. As we're moving to a DevOps world, software developers become also testers, the use case for the XML-DSL gets
more and more obsolete. The deprecation should encourage users to start developing their tests in Java and migrate their
current XML-DSL test suites. We're not going to remove the XML-DSL completely from the framework any time soon but on the
long term, this will be the case. We'll announce the final removal with at least one year of lead time for migration.
Until then, the XML-DSL will still be usable as it is but won't be extended or supported.

### Improve release process automation
To be able to release Citrus, in a dedicated release schedule, we want to invest in our release automation.
On the long term, we want to ship often and in small increments so that you don't have to wait long for your contribution,
your requested feature, enhancement or bug fix to be released. 

### Deprecate TestDesigner
To clean up the Java DSL, improve the maintainability of the framework and reduce the complexity from a users perspective,
we're going to deprecate the TestDesigner. We recommend the TestRunner for a while now and think, that the 
way it works, is the way we want to go. Therefore we'll migrate all Java samples to the TestRunner in the near future.
This will help you to get started with a new project and as well with a migration. Speaking of migration, we're planning to
merge the DSL style of the TestDesigner to the TestRunner. Our goal is to get the TestRunner DSL as close as possible to
the TestDesigner DSL so that the migration won't consume much time. In addition, we think the TestDesigner DSL is much
easier to read and write. So merging both DSL styles should make it easier for everybody to create test cases and also
helps new users to understand, how the framework works.

### Improve Logging
### Review Maven archetypes
### Stop Spring 4 Bugfix Support
### Improve documentation
### Improve Citrus project structure
### Improve test class creation
### Remove ANT support
### Create Citrus Development guide


Sven Hettwer ([@SvenHettwer](https://twitter.com/SvenHettwer))





 