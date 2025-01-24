---
layout: post
title: Cloud-native BDD testing with YAKS
short-title: YAKS BDD testing
author: Christoph Deppisch
github: christophd
categories: [blog]
---

YAKS is a testing platform that leverages Behavior Driven Development concepts for running tests on Cloud-native infrastructure.

# What is "YAKS"?

YAKS is an Open Source test automation platform to run your tests as Cloud-native resources on [Kubernetes](https://kubernetes.io/) or [OpenShift](https://www.openshift.com/). 
This means the testing tool runs your tests natively on Kubernetes and is specifically designed to verify Serverless and Microservice applications. 
A typical YAKS test uses the very same infrastructure as the System under test and exchanges data/events over different 
messaging transports (e.g. Http REST, Knative eventing, Kafka, JMS and many more).

The tests in YAKS follow the BDD (Behavior Driven Development) concepts, so you declare [Gherkin](https://cucumber.io/docs/gherkin/reference/) (Given-When-Then syntax) 
feature files and run those directly as Pod in your cluster.

YAKS leverages the Operator SDK and provides a specific operator to manage the test case resources on the cluster. 
Each time you declare a test in the form of a [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) the YAKS operator automatically takes care of preparing the 
proper runtime in order to execute the test as a Kubernetes Pod. 
The runtime uses a Java virtual machine runtime with Maven and leverages [Cucumber](https://cucumber.io/) and [Citrus](https://citrusframework.org) to run the tests.

YAKS as a framework brings a set of ready-to-use [Cucumber steps](https://cucumber.io/) so you can just start writing your feature files to verify your deployed services.

# Why would you want Cloud-native testing?

Why would you want to run tests as Cloud-native resources on the Kubernetes platform? 
Kubernetes has become a standard target platform for Serverless and Microservices architectures. 
Developing the services is different in many aspects compared to what we have done for decades.

Writing a Serverless or Microservices application for instance with [Camel K](https://camel.apache.org/projects/camel-k/) is very declarative. 
As a developer you write a Camel route and run this route as an integration via the Camel K operator directly on the cluster.

The declarative approach as well as the nature of Serverless applications make us rely on a given runtime infrastructure, and it is quite hard to run tests outside that infrastructure. 
So it is only natural to also move the verifying tests into this very same infrastructure. 
This is why YAKS brings your tests to the cloud infrastructure for integration and end-to-end testing.

Let us have a look at a sample Camel K integration. 
The integration provides a Http service and transforms incoming requests to messages sent to a Kafka topic.

*Camel K sample: http-to-kafka.groovy*
```groovy
// expose a rest endpoint that routes transformed messages to Kafka
rest().post("/greeting")
    .route()
    .transform()... // any kind of transformation
    .to("kafka:greetings")
```

Once you have written the Camel route you are ready to run the Camel K integration source within the Kubernetes infrastructure. 
It is only natural to also move the verifying tests into this very same infrastructure because you can make use of the very same messaging transports, 
databases and services (internal and external) provided in that infrastructure.

The tests are able to simulate 3rd party services or other microservices that are part of the message processing logic. 
The BDD tests describe the given context, the events to occur and the expected outcome all in one single feature file. 
This declarative testing approach is a perfect match to the concept of operators and custom resources on Kubernetes that 
is being used in so many Cloud-native services these days.

# How does it work?

YAKS provides a Kubernetes operator and a set of CRDs (custom resources) that we need to install in the cluster. 
The best way to install YAKS is to use the [OperatorHub](https://operatorhub.io/operator/yaks) or the yaks CLI tools that you can download from the [GitHub release pages](https://github.com/citrusframework/yaks/releases).

With the yaks-client binary simply run this install command:
            
```shell
$ yaks install
```

This command prepares your Kubernetes cluster for running tests with YAKS. 
It will take care of installing the YAKS custom resource definitions, setting up role permissions and creating the YAKS operator in a global operator namespace.

*Important:* You need to be a cluster admin to install custom resource definitions. 
The operation needs to be done only once for the entire cluster.

Now that the YAKS operator is up and running you can just start writing a Gherkin BDD feature file and run the test. 
YAKS brings a set of ready-to-use [Cucumber steps](https://cucumber.io/) that you can use in the feature files.

*File: http-to-kafka.feature*
```gherkin
Feature: Http to Kafka

  Background:
    Given URL: http://greeting-service-demo.svc
    Given Kafka connection
    | url   | kafka-bootstrap.server.svc:9092 |
    | topic | greetings                               |
    
  Scenario: Post greeting event
    Given HTTP request body: Hello YAKS!
    When send POST /greeting
    Then receive HTTP 201 CREATED
    And verify Kafka message with body: Hello YAKS!
```

The feature above verifies the Camel K integration to send greeting events to a Kafka topic. 
The test calls the Camel K integration with a Http POST request. 
The request invokes the service and verifies the _Http 201 CREATED_ response code. 
Also, the test verifies the Kafka message on the _greeting topic_ and compares the message content to an expected template.

The whole test uses the ready-to-use YAKS steps for handling Http and Kafka communication. 
So the tester does not have to implement this messaging transport logic.

You can run the test with:
```shell
$ yaks run http-to-kafka.feature
```

This creates a new test custom resource with the given BDD feature file. 
The YAKS operator automatically executes the test as a Pod on the cluster. 
You can review the test log output and see if the test passes.
      
```shell
$ yaks run greeting-http-to-kafka.feature
    
[...] 

[INFO]
[INFO] CITRUS TEST RESULTS
[INFO]
[INFO]  http-to-kafka.feature:6 .................. SUCCESS
[INFO] 
[INFO] TOTAL:	1
[INFO] FAILED:	0 (0.0%)
[INFO] SUCCESS:	1 (100.0%)
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] Generated test report: target/citrus-reports/citrus-test-results.html

1 Scenarios (1 passed)
2 Steps (2 passed)
0m1.711s

[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.274 s - in org.citrusframework.yaks.YaksTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 6.453 s
[INFO] Finished at: 2020-03-06T15:30:53Z
[INFO] ------------------------------------------------------------------------
Test result: Passed
```

You can also review the Pod test outcome with:
```shell
$ kubectl get tests
```

This is an example output you should get:
```shell
NAME             PHASE
http-to-kafka    Passed
```

So, what happens behind the scenes when running this test on Kubernetes?

![yaks-architecture.png](/img/assets/introducing-yaks/yaks-architecture.png)

The **yaks** tool synchronizes your test code with a Kubernetes custom resource of Kind Test. 
The resource is named `http-to-kafka` (after the file name) in the current namespace. 
So every time you run the test the custom resource is updated and executed.

The YAKS operator is the component that makes all this possible by configuring all Kubernetes resources needed for running your tests. 
The test runtime uses Cucumber and Citrus to read the feature files and run the tests in a Java virtual machine.

Now let's have a look at the predefined YAKS step implementations that you can use out of the box.

# YAKS test steps

YAKS provides several ready-to-use [Cucumber steps](https://cucumber.io/) that you can just use in your feature files. 
These steps should help you to verify applications that exchange data over various messaging transports. 
Have a look at the predefined steps we have so far:

| Module        | Description   |
| ------------- |:-------------:|
| yaks-standard | Basic steps (e.g. for logging messages to the console, test delays, settings) |
| yaks-http | Call Http REST endpoints as a client and verify the response content. Provide a Http service that receives/verifies requests and simulates response messages. |
| yaks-openapi | Import OpenAPI specifications and use defined operations and test data to call Http REST endpoints |
| yaks-jdbc | Connect to a database for updating data or verifying SQL query result sets |
| yaks-camel | Create, start and stop [Apache Camel](https://camel.apache.org/) routes as part of the test. This opens access to all **300+** Camel components for testing! |
| yaks-camel-k | Create and verify Camel K integrations as part of the test. |
| yaks-kafka | Publish/consume events on Kafka streams |
| yaks-knative | Connect with Knative eventing and leverage triggers, channels, subscriptions to exchange cloud events. |
| yaks-kubernetes | Apply resources on the Kubernetes cluster and provide services with exposed ports service simulation. |
| yaks-jms | Publish/consume messages on a JMS broker. |
| yaks-selenium | Run UI tests with a Selenium remote browser simulating user interaction on a web frontend. |

The list of ready-to-use steps is constantly growing, and you can also write your own steps and use them in a YAKS test! 
Have a look at the [examples](https://github.com/citrusframework/yaks/tree/master/examples) to see all those steps in action.

The steps provided in YAKS are implemented using the integration test framework [Citrus](https://citrusframework.org/). 
This means that you can use features like functions, validation matchers and test variables in your test.

# Demo

The following demo video shows an example of what you can do with YAKS. 
It has the YAKS operator already installed and invokes a system under test via Http verifying the response content. 
It continues to more complex scenarios where the test uses a Swagger OpenAPI specification for generating requests and test data.

[https://www.youtube.com/embed/fR-UgzvZkuA](https://www.youtube.com/embed/fR-UgzvZkuA)

# What's next

This is just the start of a new testing platform for BDD testing in Cloud-native environments! 
We are happy to receive feedback, ideas and of course contributions. 
Please add your thoughts on the GitHub repository by opening [new issues](https://github.com/citrusframework/yaks/issues) 
or even share your appreciation with a [star](https://github.com/citrusframework/yaks) on GitHub.
