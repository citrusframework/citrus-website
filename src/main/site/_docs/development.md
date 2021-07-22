---
layout: docs
title: Development
permalink: /docs/development/
---

This quickstart gets the Citrus project sources running in a few minutes, starting with initial git repository clone and 
ending with a built Citrus project ready for coding.

## Preconditions

- **[Git](http://git-scm.com/)**
  This can be either a command line client or some graphical UI. For simplicity, we assume you have the command line client installed.
- **Java 11 (or higher version)**
  You can verify the Java installation via command line with

{% highlight shell %}  
java -version
openjdk version "11.0.1" 2018-10-16
OpenJDK Runtime Environment 18.9 (build 11.0.1+13)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.1+13, mixed mode)
{% endhighlight %}
  
- **Maven 3.3.x (or higher version)**
  Download <a href="http://maven.apache.org">maven</a> and install Maven on your machine. Please verify correct version and **MAVEN_HOME** setup using following command
          
{% highlight shell %}  
mvn -version
Apache Maven 3.6.3
{% endhighlight %}

## Initial git clone

First of all we get the Citrus sources from the repository on [GitHub](http://www.github.com/). You can use the following command to do this

{% highlight shell %}  
git clone git://github.com/citrusframework/citrus.git
{% endhighlight %}

This will clone the Citrus project to the target directory **citrus**. In the following this project directory is referred 
to as **PROJECT_HOME**. For detailed instructions about the version control system git, please consult the [official git 
website](http://git-scm.com/).

## Build the Citrus artifacts

Now everything is setup properly and you can use Maven for all the rest:

{% highlight shell %}  
mvn install
{% endhighlight %}

This command runs the full Maven build lifecycle with compilation, testing, packaging and installation of all artifacts. 
You will find the freshly built Citrus JAR files in your local Maven repository. Using this new own Citrus version is 
quite simple. Just add the SNAPSHOT dependency to your projects POM like this

{% highlight shell %}  
<dependency>
  <groupId>com.consol.citrus</groupId>
  <artifactId>citrus-core</artifactId>
  <version>${citrus.version}</version>
  <scope>test</scope>
</dependency>
{% endhighlight %}

## Create IDE project files

You can easily create the project files for your favorite IDE (IntelliJ IDEA, Eclipse or Netbeans). In your **<PROJECT_HOME>** call

{% highlight shell %}
mvn idea:idea
mvn eclipse:eclipse
mvn netbeans:netbeans
{% endhighlight %}

The project files are now ready for import in your IDE. This is the preferred way for creating IDE project files in Maven. 
Please do not create IDE projects manually. Maven takes care of the whole project classpath construction.

Make sure that you have set the **M2_REPO** classpath variable set in Eclipse (or Netbeans). The variable value points to 
your local Maven repository (typically found in *C:\Documents and Settings\username\.m2\repo* or *~/.m2/repo*).

Maven ofers a suitable command to do this automatically:

{% highlight shell %}
mvn -Declipse.workspace=<path-to-eclipse-workspace> eclipse:add-maven-repo 
{% endhighlight %}

## What's next?

Have fun with Citrus!
