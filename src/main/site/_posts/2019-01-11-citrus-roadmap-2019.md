---
layout: post
title: Roadmap for 2019
short-title: Roadmap for 2019
author: Sven Hettwer
github: svettwer
categories: [blog]
---

**Happy new year everybody**

The year has just started and I am happy that we are finally ready to share the roadmap for Citrus in 2019 with you. 
As promised via [twitter](https://twitter.com/citrus_test/status/1073162157259440128), this year
*[...] will be about community, enhancing documentation and improving the ease of use of Citrus with a focus on the 
API, the tooling and the commonly used modules*. With more than ten years of history, Citrus has grown from a utility
framework in a customer's project into a full fledged message based integration testing framework with the capability of
testing complex software systems supporting a vast amount of communication protocols and messaging technologies.
Every year, the feature set and the number of available modules has been grown and because of this, Citrus has become the
powerful, reliable and flexible testing tool it is today. And this year, we want to make Citrus even more awesome!

In this post, I will go through all the points mentioned and provide some details. The roadmap only contains the major topics 
we want to approach. We will still continue to answer your questions, fix bugs and add enhancements and minor features. 
We will also create epics on GitHub for every major topic to make the progress transparent to the community. Please bear in mind that the dates
marked in the timeline are a rough estimation to give you an idea of when we want to work on a certain topic.
And as always, if you have feedback for us, just let us know!  
But before we go into the details of the roadmap, let us have a look at some other things that we are going to change.

## Release schedule
We are planning to release a new Citrus version constantly every two months starting with the release of *v2.7.9* and
*v2.8.0* this January. This will provide a constant stream of improvements and new features to you.
In addition, we are planning to release a major version of the framework once a year if required, to be able to improve
the API based on the feedback of the community and perform major version updates of our dependencies. 

## High-Priority- and Low-Priority-Modules
During the lasts months, we have monitored the download statistics from the maven central repository and recognized that
some modules are significantly more popular than others. From today on, we want to focus on the modules that are heavily
used and provide the most benefit by the community. Therefore we will divide the Citrus modules into *High-Priority* and
*Low-Priority* modules. We will invest the major part of our time into the High-Priority-Modules to improve the documentation,
samples as well as the GitHub support and also focus on bug fixes and new features. The Low-Priority-Modules on the other
hand, will take a back seat concerning these topics. If you are interested in helping us maintain any modules, feel free
to contact us and propose pull requests. We always appreciate any kind of contribution.    

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
* JMX
* Kubernetes
* Restdocs
* RMI
* Zookeeper

## Roadmap 2019

[ ![roadmap-2019.png](${context.path}/img/citrus/roadmap-2019.png)](${context.path}/img/citrus/roadmap-2019.png){:target="_blank"}

### Citrus-JDBC Improvements
Based on the feedback we received last year, we are going to improve the Citrus-JDBC functionality. This includes
output parameters in callable statements, JDBC batch statements, various data types and much more.

### Reduce/review Citrus samples
The [citrus-samples](https://github.com/citrusframework/citrus-samples) repository contains a vast amount of sample 
projects using Citrus's Java and XML DSLs. The goal of these sample projects is to demonstrate how the functionality 
should be used to new Citrus users. 
Over time the number of samples has grown so large that it is now overwhelming new uses.
That is why we want to review the Citrus samples completely and create a set that reflects the functionality of the framework
while making them easy to understand and a good point to start from. To achieve this, we will focus on samples for the
High-Priority-Modules and reduce the amount of samples for Low-Priority-Modules.

### Deprecate XML-DSL
In order to improve the maintainability of the framework and reduce the complexity from a users perspective, we are going
to deprecate the XML-DSL. The use case for the XML-DSL was to provide a way to create integration tests without any knowledge
of Java. As we are moving to a DevOps world and software developers become testers as well, the use case for the XML-DSL gets
more and more obsolete. The deprecation should encourage users to start developing their tests in Java and migrate their
current XML-DSL test suites. 
It will be some time before the XML-DSL is completely removed from the framework but going forward the focus is 
clearly on the Java DSL. 
We will announce the final removal with at least one year of lead time for migration. Until then, the XML-DSL
will still be usable as it is but it will not be extended or supported as part of the open source project.

### Improve release process automation
To be able to release Citrus, in a dedicated release schedule, we want to invest in our release automation.
In the long term we want to ship often and in small increments. In the future you will not have to wait as long for your 
contribution, requested feature, enhancement or bug fix to be released.

### Deprecate TestDesigner
To clean up the Java DSL, improve the maintainability of the framework and reduce the complexity from a users perspective,
we are going to deprecate the TestDesigner. We have been recommending the TestRunner for some time now and think, that the 
way it works is superior to the TestDesigner. Therefore we will migrate all Java samples to the TestRunner in the near future.
This will help you to get started with a new project and as well with a migration. Speaking of migration, we are planning to
merge the DSL style of the TestDesigner to the TestRunner. Our goal is to get the TestRunner DSL as close as possible to
the TestDesigner DSL so that the migration will not consume much time. In addition, we think the TestDesigner DSL comes with
some advantages and is much easier to read, write and understand. So merging both DSL styles should make it easier for
everybody to create test cases and also helps new users to understand how the framework works. In v3.0.0, the
TestDesigner will finally be removed from the framework.

### Improve Logging
To make it easier to detect the root cause of failing tests, we will review the current logging concept to provide you
the information you need when something goes wrong. This should help you to identify issues with your test case
or your system under test as efficient as possible.

### Review Maven archetypes
Maven archetypes are a good point to start from. No matter if you want to start a new project or if you are new to the
framework, archetypes help you to generate the required boilerplate code. With a review of the Citrus archetypes, we want
to ensure it is as easy as possible for you to create the infrastructure required by the framework.

### Stop Spring 4 Bugfix Support
As you may have noticed that Citrus switches to Spring 5 with the v2.8.0 release. Even if the public API of Citrus remains 
untouched by this change, it might be the case that some users rely on Spring 4 in other components of their test
suite. Therefore we will provide a Spring 4 branch of the framework with bug fixes but without new features until mid 2019.
The Spring 4 releases with continue within the v2.7.x release family.

### Improve documentation
Citrus already comes with a detailed and explanatory documentation, giving insides into the framework and also into the
integrated communication technology. We will review the documentation's structure and content to make it easier to navigate
and focus on the features of the framework.

### Improve Citrus project structure
To improve the maintainability of the framework, we are going to review some structures that have established over the last years.
This mainly concerns the packages and module structure. In addition we want to make it easier to get started with Citrus
by reviewing the way we have setup our dependencies. The goal is to reduce the amount of required dependencies making the
configuration clearer and easier to understand for new users. The changes to the package structure will be released
in v3.0.0 by the end of the year.

### Improve test class creation
As we are going to review and clean up our API in 2019, we will also review the way test classes are created. Again the goal  
here is to make the setup easier to understand and more expressive to the user. The changes to the test class creation
will be released in v3.0.0 by the end of the year.

### Remove ANT support
As maven and gradle are the most prevalent build systems on the market, we are going to remove the ant support from the framework.
This mainly concerns the ANT test actions. The support of ANT will be stopped with v3.0.0 by the end of the year.

### Create Citrus Development guide
To encourage the community to contribute to the project and to provide recipes for extending the framework from a developers
perspective, we want create a development guide that helps you to start developing Citrus. The idea here is to make this
a living document with small incremental additions over time.

## Thank you    
At this point thanks a lot to all the contributors to the framework! We highly appreciate every contribution from the
community! Whether it be through bug reports, ideas for new features, pull requests, general feedback or anything else, you help us
to improve Citrus step by step and that is awesome!

We are looking forward to 2019! 

Sven Hettwer ([@SvenHettwer](https://twitter.com/SvenHettwer))  
Citrus Maintainer and Senior Software Engineer @ ConSol GmbH