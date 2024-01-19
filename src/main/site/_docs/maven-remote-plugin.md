---
layout: docs
title: Maven remote plugin
permalink: /docs/maven-remote-plugin/
---

The Citrus Maven Remote plugin can be used for packaging all Citrus tests into a **test-server** and for accessing this
test-server via a remote API call. At present the following remote API calls are supported:
- Execute the citrus-integration tests.
- Poll the test-server, checking whether the test execution has completed.

This provides an alternative means of executing tests to the usual surefire and failsafe maven plugins 
that execute the tests locally within the scope of the maven build. 
The main benefit of using this approach however is that the Citrus tests can be built and run inside of a 
docker container. This makes it easier to execute the tests and mock the endpoints used by the system under test that 
may otherwise not be possible in certain containerized environments.  

## System Requirements

The following specifies the minimum requirements to run this Maven plugin:

| Library | Version |
|:-------:|:-------:|
| Maven | 3.3 |
| JDK | 1.8 |
| Memory | No minimum requirement. |
| Disk Space | No minimum requirement. |

## Usage

You should specify the version in your project's plugin configuration:

{% highlight xml %}  
<project>
  ...
  <build>
    <!-- To define the plugin version in your parent POM -->
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>org.citrusframework</groupId>
          <artifactId>citrus-remote-maven-plugin</artifactId>
          <version>${citrus.version}</version>
        </plugin>
        ...
      </plugins>
    </pluginManagement>
    <!-- To use the plugin goals in your POM or parent POM -->
    <plugins>
      <plugin>
        <groupId>org.citrusframework</groupId>
        <artifactId>citrus-remote-maven-plugin</artifactId>
        <configuration>
          ...
        </configuration>
      </plugin>
      ...
    </plugins>
  </build>
  ...
</project>
{% endhighlight %}  

## Goals

| Goal | Description |
|:-------:|:-------:|
| citrus-remote:test-jar | Assembles an executable JAR file containing the test-server and all test cases. |
| citrus-remote:test-war | Assembles a WAR file that contains all test cases that can be deployed in any servlet container. |
| citrus-remote:test | Triggers the execution of the test cases on the remote test-server |
| citrus-remote:verify | Checks the results of the test execution on the remote test-server, waiting if necessary for the completion of the test execution. |
| citrus-remote:help | Display help information on citrus-remote-maven-plugin. To display parameter details call: _mvn citrus-remote:help -Ddetail=true -Dgoal=\<goal-name>_ |

## citrus-remote:test-jar

### Full name

    org.citrusframework:citrus-remote-maven-plugin:${citrus.version}:test-jar

### Description

Packages all integration tests into a JAR file that can be executed. 

After creation the JAR file, which contains a test-server, can be started manually using: *java -jar \<jar-file>* 

### Parameters

| Name | Type | Description |
|:-------:|:-------:|:-------:|
| finalName | String | The prefix used for the executable jar filename. The format used is \<**finalName**>-citrus-tests.jar. Default value is: \${project.build.finalName}. |
| mainClass | String | The class that is executed when the jar file is run. Default value is: org.citrusframework.remote.CitrusRemoteServer
| skip | Boolean | Skip the creation of the executable JAR file. Default value is: false. Can also be set using \${citrus.remote.plugin.skip} |
| skipTestJar | Boolean | Skip the creation of the executable JAR file. Default value is: false. Can also be set using \${citrus.skip.test.jar} |
| testJar | Embedded |  The test jar configuration. Refer to the [Test JAR configuration](#test-jar-configuration) for details. |
| assembly | Embedded | The assembly configuration. Refer to the [Assembly configuration](#assembly-configuration) for details. |

## citrus-remote:test-war

### Full name

    org.citrusframework:citrus-remote-maven-plugin:${citrus.version}:test-war

### Description

Packages all integration tests into a WAR file. 

After creation the WAR file can be deployed in any standard servlet container. 

### Parameters

| Name | Type | Description |
|:-------:|:-------:|:-------:|
| finalName | String | The prefix used for the WAR filename. The format used is \<**finalName**>-citrus-tests.war. Default value is: \${project.build.finalName}. |
| skip | Boolean | Skip the creation of the WAR file. Default value is: false. Can also be set using \${citrus.remote.plugin.skip} |
| skipTestWar | Boolean | Skip the creation of the war file. Default value is: false. Can also be set using \${citrus.skip.test.war} |
| testJar | Embedded |  The test jar configuration. Refer to the [Test JAR configuration](#test-jar-configuration) for details. |
| assembly | Embedded | The assembly configuration. Refer to the [Assembly configuration](#assembly-configuration) for details. |

## citrus-remote:test

### Full name

    org.citrusframework:citrus-remote-maven-plugin:${citrus.version}:test

### Description

Triggers the execution of the citrus test by performing a remote API call to the test-server. 

### Parameters

| Name | Type | Description |
|:-------:|:-------:|:-------:|
| report | Embedded | The report configuration. Refer to the [Report configuration](#report-configuration) for details. |
| run | String | The run configuration. Refer to the [Run configuration](#run-configuration) for details. |
| server | String | The server configuration. Refer to the [Server configuration](#server-configuration) for details. |
| skip | Boolean | Skip the creation of the executable jar file. Default value is: false. Can also be set using \${citrus.remote.plugin.skip} |
| skipRun | Boolean | Skip the running of the tests. Default value is: false. Can also be set using \${citrus.remote.skip.test} |
| timeout | Integer | Http connect timeout in milliseconds. Default value is: 60000. Can also be set using \${citrus.remote.plugin.timeout} |

## citrus-remote:verify

### Full name

    org.citrusframework:citrus-remote-maven-plugin:${citrus.version}:verify

### Description

Checks the results of the test-execution. If any tests fail then verify MOJO will fail causing the maven build to fail.

### Parameters

| Name | Type | Description |
|:-------:|:-------:|:-------:|
| failIfNoTests | Boolean | Fail build if not tests were executed. Default value is: true. Can also be set using \${citrus.remote.failIfNoTests} |
| report | Embedded | The report configuration. Refer to the [Report configuration](#report-configuration) for details. |
| skip | Boolean | Skip the creation of the executable jar file. Default value is: false. Can also be set using \${citrus.remote.plugin.skip} |

## Assembly configuration
<a name="assembly-configuration"></a>
The citrus-remote-maven-plugin provides out of the box assembly descriptors for building the war and executable-jar 
files.

**default executable-jar assembly:**
```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  <id>citrus-tests</id>
  <formats>
    <format>jar</format>
  </formats>
  <includeBaseDirectory>false</includeBaseDirectory>
  <dependencySets>
    <dependencySet>
      <outputDirectory>/</outputDirectory>
      <useProjectArtifact>false</useProjectArtifact>
      <useProjectAttachments>true</useProjectAttachments>
      <unpack>true</unpack>
      <scope>test</scope>
    </dependencySet>
  </dependencySets>
</assembly>
```

**default war-assembly:**
```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  <id>citrus-tests</id>
  <formats>
    <format>war</format>
  </formats>
  <includeBaseDirectory>false</includeBaseDirectory>
  <dependencySets>
    <dependencySet>
      <outputDirectory>/WEB-INF/lib</outputDirectory>
      <useProjectArtifact>false</useProjectArtifact>
      <useProjectAttachments>true</useProjectAttachments>
      <unpack>false</unpack>
      <scope>test</scope>
      <excludes>
        <exclude>org.eclipse.jetty:jetty-webapp</exclude>
        <exclude>org.eclipse.jetty.websocket:websocket-server</exclude>
        <exclude>org.eclipse.jetty.websocket:websocket-servlet</exclude>
      </excludes>
    </dependencySet>
  </dependencySets>
  <fileSets>
    <fileSet>
      <directory>\${project.basedir}/src/test/resources/WEB-INF</directory>
      <outputDirectory>WEB-INF</outputDirectory>
      <includes>
        <include>**/*</include>
      </includes>
    </fileSet>
  </fileSets>
</assembly>
```

You can overwrite the defaults by configuring your own assembly. For details on how to create an assembly refer to the 
[maven-assembly-plugin](http://maven.apache.org/plugins/maven-assembly-plugin). To overwrite the default use one of the 
following approaches:
 
### Assembly configuration using a file
```xml
<assembly>
    <descriptor>
        <file>\${project.basedir}/citrus-assembly/citrus-war-assembly.xml</file>
    </descriptor>
</assembly>
```

### Assembly configuration using a reference
```xml
<assembly>
    <descriptor>
        <ref>my-assembly</ref>
    </descriptor>
</assembly>
```
and then make sure to add the my-assembly.xml file under my-assembly-descriptor
```
+-- src
|   `-- main
|       `-- resources
|           `-- assemblies
|               `-- myassembly.xml 
```

### Assembly configuration using an inline assembly
 
```xml
<assembly>
    <descriptor>
        <inline>
              <formats>
                <format>war</format>
              </formats>
              <includeBaseDirectory>false</includeBaseDirectory>
        </inline>
    </descriptor>
</assembly>
```

## Server configuration
<a name="server-configuration"></a>
The remote-maven-plugin uses a remote API call to trigger the test execution or to verify the results. The url of the 
test-server can be configured as shown below:  
 
```xml
<server>
    <url>http://citrus.remote-test-server.org/integration-citrus-test</url>
</server>
``` 

## Report configuration
<a name="report-configuration"></a>
Report configuration such as output directory and file names.

The default configuration can be overwritten as shown below:

```xml
<report>
    <directory>citrus-reports</directory>
    <summaryFile>citrus-summary.xml</summaryFile>
    <htmlReport>true</htmlReport>
    <saveReportFiles>true</saveReportFiles>
</report>
``` 

| Name | Type | Description |
|:-------:|:-------:|:-------:|
| directory | String | The report directory. Default value is: _citrus-reports_. Can also be set using \${citrus.report.directory} |
| summaryFile | String | The summary file name. Default value is: _citrus-summary.xml_. Can also be set using \${citrus.report.summary.file} |
| htmlReport | Boolean | Enable/disable HTML report generation. Default value is: _true_. Can also be set using \${citrus.report.html}  |
| saveReportFiles | Boolean | Get reporting files from server and save them in report output directory. Default value is: _true_. Can also be set using \${citrus.report.save.files}  |

## Run configuration
<a name="run-configuration"></a>
Run configuration for test execution on remote server.

This can be configured as shown below:
```xml
<run>
    <classes>
        <class>com.consol.smoketests.IntegrationTestIT</class>
    </classes>
    <packages>
        <package>com.consol.integrationtests</package>
    </packages>
    <includes>
        <include>^.*IT$</include>
    </includes>
    <async>true</async>
    <pollingInterval>10000</pollingInterval>
    <systemProperties>
        <spring.profiles.active>TEST</spring.profiles.active>
    </systemProperties>
</run>
``` 

| Name | Type | Description |
|:-------:|:-------:|:-------:|
| classes | String | A list of citrus test classes to be executed |
| packages | String | A list of packages to use to locate the tests to be executed |
| includes | String | A list of patterns for locating tests to be executed |
| async | Boolean | If true the test execution does not wait for all tests to complete before returning. Default value is: false (wait for tests to complete). Can also be set using \${citrus.remote.run.async} |
| pollingInterval | Integer | The polling interval in milliseconds when checking for test execution completion. This is required when the tests are executed asynchronously (async=true). Default value is: 10000. Can also be set using \${citrus.remote.run.polling.interval} |
| systemProperties |String | The list of systemProperties to set when starting the test execution. |

## Test JAR configuration
<a name="test-jar-configuration"></a>
The testJar configuration can be used for configuring which tests should be included in or excluded from the assembled 
JAR or WAR file.
The default configuration:
- includes all tests 
- uses the classifier _tests_ as a suffix in the JAR or WAR filename (e.g. integration-citrus-tests.war)   

The default configuration can be overwritten as shown below:
  
```xml
<testJar>
    <classifier>tests</classifier>
    <includes>
        <include>^.*IT$</include>
    </includes>
    <excludes>
        <exclude>^.*Skip_IT$</exclude>    
    </excludes>
</testJar>
``` 

The files to be included or excluded can be specified as fileset patterns which are relative to the input directory 
whose contents is being packaged into the test JAR.

