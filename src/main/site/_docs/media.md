---
layout: docs
title: Media
permalink: /docs/media/
---

Find below videos, conference talks and live demo sessions about Citrus and YAKS.

## Quarkus Insights - Citrus Integration Test Framework
### Episode #149

<iframe width="560" height="315" src="https://www.youtube.com/embed/0wvWd2woBJ8" frameborder="0" allowfullscreen></iframe>

In this podcast episode we discuss the Citrus testing framework and how it can help to verify Quarkus applications in an automated integration test.

## Testing Kamelets - Verify event sources and sinks with YAKS
### Presentation from ApacheCon 2021

<iframe width="560" height="315" src="https://www.youtube.com/embed/mIfmMFOgdbI" frameborder="0" allowfullscreen></iframe>

Kamelets represent Camel K route snippets that act as standardized event sources and sinks in an event driven architecture. 
Kamelets provide a simple interface that hides the individual implementation details and allows the user to connect to external 
systems with simply providing some required properties. 
YAKS is a test automation tool that provides special support for Camel K, Kamelets and other technologies such as Knative, 
Apache Kafka, OpenAPI 3, Http REST, JMS, and many more. 
The presentation shows how to write automated integration tests for Kamelets with YAKS in order to run those tests as part of Kubernetes. 
The talk begins with a short introduction into the framework concepts and illustrates the general test automation for event sources
and sinks in the form of examples and live demos. 
In the end the audience should be able to accomplish automated integration tests for custom Kamelets that bind to messaging 
technologies such as Knative, Http, and Apache Kafka.

## Testing Camel K integrations with Cloud Native BDD
### Presentation from ApacheCon 2020

<iframe width="560" height="315" src="https://www.youtube.com/embed/aYLLtj6TdjM" frameborder="0" allowfullscreen></iframe>

Apache Camel K is a lightweight integration platform built from Apache Camel. 
Integrations built with Camel K run natively on Kubernetes and are specifically designed for serverless architectures.
With the declarative nature in Camel K users can instantly run integration code written in Camel DSL on their preferred cloud. 
The presentation outlines typical integration scenarios with Camel K and shows how to write automated tests for these enterprise integrations. 
The session covers classical service provider/consumer scenarios with common messaging protocols (e.g. REST, JMS, Kafka) 
as well as more complex integrations with data access and 3rd party Saas services included. 
The tests itself will also be Cloud Native citizens and make use of Behavior Driven Development concepts.

## Citrus TestTalks Interview
### Integration Automation Using Citrus Testing Framework

<a href="https://testguild.com/podcast/automation/190-integration-automation-using-citrus-testing-framework-christoph-deppisch/" target="_blank"><img src="/img/assets/testtalks-interview/thumbnail.png" alt="TestTalks Interview"></a>

Copyright by Joe Colantonio, Test Guild LLC

Joe Colantonio talks to Christoph Deppisch about automate integration tests for pretty much any messaging protocol or data format.
So if you’re looking to expand your automation efforts beyond lame, flaky, UI-based automated tests, this episode is for you.
Listen to this "testtalks" episode at <a href="https://testguild.com/podcast/automation/190-integration-automation-using-citrus-testing-framework-christoph-deppisch/" target="_blank">https://testguild.com/podcast</a>
    
## BDD Integration with Citrus & Cucumber
### Citrus "Tools in action" session @DevoxxBE 2017

<iframe width="560" height="315" src="https://www.youtube.com/embed/X76Wz0MNdrc" frameborder="0" allowfullscreen></iframe>

The concept of behavior driven development (BDD) is quite simple. 
Business analysts and domain experts describe how the application should behave using Gherkin (given-when-then) feature stories.
Developers glue those specifications to automated tests. 
Can we use this approach when testing the integration of services, too? 
The talk gives answers with a combination of the frameworks Cucumber and Citrus.
After a short introduction to both frameworks we discuss the concepts and see some code examples that demonstrate how 
the behavior driven approach fits to testing the messaging integration with Http REST, JMS and mail communication.
At the end we will have an automated integration test with BDD and almost no glue code to write manually.
    
## Citrus & Cucumber BDD
### Citrus "Tools in action" session @DevoxxUS 2017

<iframe width="560" height="315" src="https://www.youtube.com/embed/jNgROZjI98Y" frameborder="0" allowfullscreen></iframe>

The concept of behavior driven development (BDD) is quite simple. 
Business analysts and domain experts describe how the application should behave using Gherkin (given-when-then) feature stories. 
Developers glue those specifications to automated tests. 
The talk combines the frameworks Cucumber and Citrus and gives some code examples that demonstrate how the behavior driven approach fits to
testing the messaging integration with Http REST, JMS and mail.

## Testing Microservices
### Citrus "Tools in action" session @DevoxxBE 2015

<iframe width="560" height="315" src="https://www.youtube.com/embed/FPgXJveaLTo" frameborder="0" allowfullscreen></iframe>

Citrus integrates with frameworks like Apache Camel, Arquillian, Kubernetes and Docker in order to provide automated integration 
testing of Microservice applications. 
The tools in action session gives a brief introduction to the Citrus framework and shows code samples for a complete integration 
test scenario in a Microservices environment.
