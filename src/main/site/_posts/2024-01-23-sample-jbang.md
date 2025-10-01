---
layout: sample
title: JBang sample
name: sample-jbang
image: /img/icons/jbang.png
folder: common
group: demo
description: Prototype Citrus tests in minutes with JBang
categories: [samples]
permalink: /samples/jbang/
---

You can easily create and run Citrus tests with [JBang](https://www.jbang.dev/).
The JBang tool support in Citrus is described in more detail in [reference guide][1]

Running Citrus via JBang does not require any project setup which is a fantastic match for fast prototyping of integration tests.
The JBang command will automatically set up everything you need to run the Citrus test.
This means you can run your test case sources directly from your command line.

To initialize a test you can run the script `citrus@citrusframework/citrus` with the `init` command as follows:

_Initialize my-test.yaml_
```shell
jbang citrus@citrusframework/citrus init my-test.yaml
```

The command above uses the JBang catalog `citrus@citrusframework/citrus` located on the [Citrus GitHub repository](https://github.com/citrusframework/citrus).
JBang will automatically resolve all dependencies and execute the command line script tool.
This initializes the Citrus test file.
You will find the created test source file in the current directory.

_my-test.yaml_
```yaml
name: my-test
author: Citrus
status: FINAL
description: Sample test in YAML
variables:
  - name: message
    value: Citrus rocks!
actions:
  - echo:
      message: "${message}"
```

The JBang script is able to initialize any supported Citrus test domain specific language `.java`, `.xml`, `.yaml`, `.groovy` or `.feature`.

You can now run this test source file without any prior project setup using Citrus JBang:

_Run my-test.yaml_
```shell
jbang citrus@citrusframework/citrus run my-test.yaml
```

The command output will be like this:

_Output_
```shell
[main] citrusframework.testng.TestNGEngine : Running test source my-test.yaml
[main] org.testng.internal.Utils           : [TestNG] Running:
[main] rusframework.report.LoggingReporter :        .__  __                       
[main] rusframework.report.LoggingReporter :   ____ |__|/  |________ __ __  ______
[main] rusframework.report.LoggingReporter : _/ ___\|  \   __\_  __ \  |  \/  ___/
[main] rusframework.report.LoggingReporter : \  \___|  ||  |  |  | \/  |  /\___ \ 
[main] rusframework.report.LoggingReporter :  \___  >__||__|  |__|  |____//____  >
[main] rusframework.report.LoggingReporter :      \/                           \/
[main] rusframework.report.LoggingReporter : 
[main] rusframework.report.LoggingReporter : C I T R U S  T E S T S  ${citrus.version}
[main] rusframework.report.LoggingReporter : 
[main] rusframework.report.LoggingReporter : ------------------------------------------------------------------------
[main] .citrusframework.actions.EchoAction : Citrus rocks!
[main] rusframework.report.LoggingReporter : 
[main] rusframework.report.LoggingReporter : TEST SUCCESS my-test (org.citrusframework)
[main] rusframework.report.LoggingReporter : ------------------------------------------------------------------------
[main] rusframework.report.LoggingReporter : 
[main] rusframework.report.LoggingReporter : 
[main] rusframework.report.LoggingReporter : CITRUS TEST RESULTS
[main] rusframework.report.LoggingReporter : 
[main] rusframework.report.LoggingReporter : SUCCESS (     3 ms) my-test
[main] rusframework.report.LoggingReporter : 
[main] rusframework.report.LoggingReporter : TOTAL:		1
[main] rusframework.report.LoggingReporter : SUCCESS:	1 (100.0%)
[main] rusframework.report.LoggingReporter : FAILED:		0 (0.0%)
[main] rusframework.report.LoggingReporter : PERFORMANCE:	0 ms
[main] rusframework.report.LoggingReporter : 
[main] rusframework.report.LoggingReporter : ------------------------------------------------------------------------

===============================================
Default Suite
Total tests run: 1, Passes: 1, Failures: 0, Skips: 0
===============================================
```

## Install Citrus JBang app

For a more convenient command line usage you can install Citrus as a JBang app.

_Install Citrus as JBang app_
```shell
jbang trust add https://github.com/citrusframework/citrus/
jbang app install citrus@citrusframework/citrus
```

Now you can just call `citrus` and create and run tests with Citrus JBang.

_Run my-test.yaml_
```shell
citrus run my-test.yaml
```

## Run tests

You can directly run test sources with Citrus JBang.
This includes test sources written in Java (`.java`), XML (`.xml`), YAML (`.yaml`), Groovy (`.groovy`) or as a Cucumber Gherkin feature file (`.feature`).

### Java test sources

_Initialize MyTest.java_
```shell
citrus init MyTest.java
```

_MyTest.java_
```java
import org.citrusframework.TestCaseRunner;
import org.citrusframework.annotations.CitrusResource;

import static org.citrusframework.actions.CreateVariablesAction.Builder.createVariables;
import static org.citrusframework.actions.EchoAction.Builder.echo;

public class MyTest implements Runnable {

    @CitrusResource
    TestCaseRunner t;

    @Override
    public void run() {
        t.given(
            createVariables().variable("message", "Citrus rocks!")
        );

        t.then(
            echo().message("${message}")
        );
    }
}
```

_Run MyTest.java_
```shell
citrus run MyTest.java
```

### XML test sources

_Initialize my-test.xml_
```shell
citrus init my-test.xml
```

_my-test.xml_
```xml
<test name="EchoTest" author="Christoph" status="FINAL" xmlns="http://citrusframework.org/schema/xml/testcase"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://citrusframework.org/schema/xml/testcase http://citrusframework.org/schema/xml/testcase/citrus-testcase.xsd">
  <description>Sample test in XML</description>
  <variables>
    <variable name="message" value="Citrus rocks!"/>
  </variables>
  <actions>
    <echo message="${message}"/>
  </actions>
</test>
```

_Run my-test.xml_
```shell
citrus run my-test.xml
```

### YAML test sources

_Initialize my-test.yaml_
```shell
citrus init my-test.yaml
```

_my-test.yaml_
```yaml
name: EchoTest
description: "Sample test in YAML"
variables:
  - name: "message"
    value: "Citrus rocks!"
actions:
  - echo:
    message: "${message}"
```

_Run my-test.yaml_
```shell
citrus run my-test.yaml
```

### Groovy test sources

_Initialize my-test.groovy_
```shell
citrus init my-test.groovy
```

_my-test.groovy_
```groovy
import static org.citrusframework.actions.EchoAction.Builder.echo

name "EchoTest"
description "Sample test in Groovy"

variables {
  message="Citrus rocks!"
}

actions {
  $(echo().message('${message}'))
}
```

_Run my-test.groovy_
```shell
citrus run my-test.groovy
```

### Cucumber feature sources

_Initialize my-test.feature_
```shell
citrus init my-test.feature
```

_my-test.feature_
```gherkin
Feature: EchoTest

  Background:
    Given variables
    | message | Citrus rocks! |

  Scenario: Print message
    Then print '${message}'
```

_Run my-test.feature_
```shell
jbang --deps org.citrusframework:citrus-cucumber-all:4.9.0-SNAPSHOT citrus run my-test.feature
```

_NOTE:_ Many of the predefined Cucumber steps (e.g. `Then print '<message>'`) in Citrus are provided in a separate Citrus child project called [YAKS](https://github.com/citrusframework/yaks).
You need to add additional project dependencies for that steps to be loaded as part of the JBang script.
The `--deps` option adds dependencies using Maven artifact coordinates.
You may add the additional modules to the `jbang.properties` as described in the next section.

## Additional JBang dependencies

Citrus JBang comes with a set of default dependencies that makes the scripts run as tests.

The default modules that you can use in Citrus JBang are:

* org.citrusframework:citrus-base
* org.citrusframework:citrus-jbang-connector
* org.citrusframework:citrus-groovy
* org.citrusframework:citrus-xml
* org.citrusframework:citrus-yaml
* org.citrusframework:citrus-http
* org.citrusframework:citrus-validation-json
* org.citrusframework:citrus-validation-xml

This enables you to run Java, YAML, XML, Groovy tests out of the box.
In case your tests uses an additional feature from the Citrus project you may need to add the module so JBang can load the dependency at startup.

The easiest way to do this is to create a `jbang.properties` file that defines the additional dependencies:

_jbang.properties_
```properties
# Declare required additional dependencies
run.deps=org.citrusframework:citrus-camel:${citrus.version},\
org.citrusframework:citrus-testcontainers:${citrus.version},\
org.citrusframework:citrus-kafka:${citrus.version}
```

The file above adds the modules `citrus-camel`, `citrus-testcontainers` and `citrus-kafka` so you can use them in your JBang Citrus test source.

The `jbang.properties` file may be located right next to the test source file or in your user home directory for global settings.

_IMPORTANT:_ In case you want to run Cucumber BDD Gherkin feature files and use the predefined steps included in the [YAKS](https://github.com/citrusframework/yaks) project,
you need to add this YAKS runtime dependency accordingly: `org.citrusframework.yaks:yaks-standard:0.20.0`

## Run from clipboard

You can run tests from your current clipboard.
Just use the file name `clipboard.xxx` where the file extension defines the type of the test source (`.java`, `.yaml`, `.xml`, `.groovy`, `.feature`).

_Run YAML test from Clipboard_
```shell
citrus run clipboard.yaml
```

## List tests

The `ls` command lists all running Citrus tests.
These tests may be started

_List running tests_
```shell
citrus ls
```

_Command output_
```shell
PID   NAME         STATUS  AGE 
19201 my-test.yaml Running 20s
```

 [1]: https://citrusframework.org/citrus/reference/html#runtime-jbang
