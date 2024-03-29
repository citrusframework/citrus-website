---
layout: docs
title: Setup with Gradle
permalink: /docs/setup-gradle/
---

This quickstart shows you how to setup a new Citrus project with Gradle. After that you will be able to get Citrus tests running 
within minutes. You can find the project sources on GitHub [citrus-samples/sample-gradle](https://github.com/citrusframework/citrus-samples/blob/master/sample-gradle).

### Preconditions

You need following software on your computer, in order to use the Citrus Framework:

* **Java 17+**
  Installed JDK plus JAVA_HOME environment variable set
  up and pointing to your Java installation directory. Used to compile and build the Citrus code.

* **Gradle 2.13+**
  Citrus tests will be executed with the [Gradle](https://gradle.org/) build tool.

* **Java IDE** (optional)
  A Java IDE will help you to manage your Citrus project (e.g. creating
  and executing test cases). You can use the Java IDE that you like best like Eclipse or IntelliJ IDEA.

Citrus uses Maven internally for building software. But of course you can also integrate the Citrus tests in a Gradle
project. Since Citrus tests are nothing but normal JUnit or TestNG tests, integration into your Gradle build is very easy.

### Gradle project

First, we create a new Java project called *citrus-sample*. There are multiple ways to get started with a Gradle project. I personally
prefer to use my Java IDE (IntelliJ) for generating a basic Gradle project setup. Of course there are lots of Gradle project start samples out there.
And summing up the Gradle project structure is pretts simple so you could also create this manually. Here is the basic project structure that we
are going to use in this quickstart.

    citrus-sample
      |   + src
      |   |   + main
      |   |    |   + java
      |   |    |   + resources
      |   |   + test
      |   |    |   + java
      |   |    |   + resources
    build.gradle
    settings.gradle

The Gradle build configuration is done in the **build.gradle** and **settings.gradle** files. Here we define the project name and the project version.

{% highlight shell %}
rootProject.name = 'citrus-sample-gradle'
group 'org.citrusframework.samples'
version '${citrus.version}'
{% endhighlight %}
    
Now, since Citrus libraries are available on Maven central repository, we add these repositories so Gradle knows how to download the required
Citrus artifacts.    

{% highlight shell %}
repositories {
    mavenCentral()
}
{% endhighlight %}
    
Now lets move on with adding the Citrus libraries to the project.

{% highlight shell %}
dependencies {
    testCompile group: 'org.citrusframework', name: 'citrus-base', version: '${citrus.version}'
    testCompile group: 'org.citrusframework', name: 'citrus-jms', version: '${citrus.version}'
    testCompile group: 'org.citrusframework', name: 'citrus-http', version: '${citrus.version}'
    testCompile group: 'org.testng', name: 'testng', version: '${testng.version}'
    [...]
}
{% endhighlight %}
    
This enables the Citrus support for the project, so we can use the Citrus classes and APIs. We decided to use TestNG unit test library.
    
{% highlight shell %}
test {
    useTestNG()
}
{% endhighlight %}

> Warning: This tutorial uses the TestNG unit test library. If you want to use JUnit, you will have to remove the above lines from your build.gradle file.
    
Of course JUnit is also supported. This is all for build configuration settings. We can move on to writing some Citrus integration tests. The Java test classes
usually go to the **src/test/java** directory.

Lets write a simple Citrus test case in Java and save it to the **src/test/java** folder in package **org.citrusframework.samples**. 

{% highlight java %}
import org.citrusframework.annotations.CitrusTest;
import org.citrusframework.channel.ChannelEndpoint;
import org.citrusframework.dsl.testng.TestNGCitrusTestDesigner;
import org.citrusframework.message.MessageType;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.test.context.ContextConfiguration;
import org.testng.annotations.Test;

@ContextConfiguration(classes = { EndpointConfig.class })
public class MessagingTest extends TestNGCitrusTestDesigner {

    @Autowired
    private ChannelEndpoint testChannelEndpoint;

    @Test
    @CitrusTest
    public void testMessaging() {
        echo("Test simple message send and receive");

        send(testChannelEndpoint)
            .messageType(MessageType.PLAINTEXT)
            .payload("Hello Citrus!");

        receive(testChannelEndpoint)
            .messageType(MessageType.PLAINTEXT)
            .payload("Hello Citrus!");

        echo("Successful send and receive");
    }
}
{% endhighlight %}

This sample uses pure Java code for both Citrus configuration and tests. The
Citrus TestNG test uses a context configuration annotation.

{% highlight java %}
@ContextConfiguration(classes = { EndpointConfig.class })
{% endhighlight %}

This tells Spring to load the configuration from the Java class ***EndpointConfig***.

{% highlight java %}
public class EndpointConfig {
    @Bean
    public ChannelEndpoint testChannelEndpoint() {
        ChannelEndpointConfiguration endpointConfiguration = new ChannelEndpointConfiguration();
        endpointConfiguration.setChannel(testChannel());
        return new ChannelEndpoint(endpointConfiguration);
    }
    
    @Bean
    private MessageChannel testChannel() {
        return new MessageSelectingQueueChannel();
    }
}
{% endhighlight %}
    
In the configuration class we are able to define Citrus components for usage in tests. As usual
we can autowire the endpoint components as Spring beans in the test cases.

{% highlight java %}
@Autowired
private ChannelEndpoint testChannelEndpoint;
{% endhighlight %}
        
### Run

The sample application uses Gradle as build tool. So you can use the Gradle wrapper to compile, package and test the
sample with Gradle build.
 
{% highlight shell %}
gradlew clean build
{% endhighlight %}
    
This executes all Citrus test cases during the build and you will see Citrus performing some integration test logging output.
After the tests are finished build is successful and you are ready to go for writing some tests on your own.

If you just want to execute all tests you can call

{% highlight shell %}
gradlew clean check
{% endhighlight %}

Of course, you can also start the Citrus tests from your favorite IDE.
Just start the Citrus test using the Gradle integration in IntelliJ, Eclipse or Netbeans.

So now you are ready to use Citrus! Write test cases and add more logic to the test project. Have fun with it!
