---
layout: sample
title: OpenAPI sample
name: sample-swagger
image: /img/icons/openapi.png
folder: samples-http
group: endpoints
description: Auto generate tests from Open API
categories: [samples]
permalink: /samples/openapi/
---

This sample uses Swagger Open API specification to generate tests for the todo app system under test. You can read more about the 
Citrus test generation features in [reference guide][1]

Objectives
---------

The [todo-list](../todo-app/README.md) sample application manages todo entries. The application provides a REST web API 
for adding new entries and listing all entries. This API is specified using Swagger ()

The sample API specification looks like this.

```json
{
  "swagger": "2.0",
  "info": {
    "description": "REST API for todo application",
    "version": "2.0",
    "title": "TodoList API",
    "license": {
      "name": "Apache License Version 2.0"
    }
  },
  "host": "localhost:8080",
  "basePath": "/",
  "paths": {
    "/api/todolist": {
      "get": {
        "summary": "List todo entries",
        "description": "Returns all available todo entries.",
        "operationId": "listTodoEntries",
        "consumes": [
          "application/json"
        ],
        "produces": [
          "*/*"
        ],
        "responses": {
          "200": {
            "description": "OK",
            "schema": {
              "type": "array",
              "items": {
                "$ref": "#/definitions/TodoEntry"
              }
            }
          }
        }
      },
      "post": {
        "tags": [
          "todo-list-controller"
        ],
        "summary": "Add todo entry",
        "description": "Adds new todo entry.",
        "operationId": "addTodoEntry",
        "consumes": [
          "application/json"
        ],
        "produces": [
          "*/*"
        ],
        "parameters": [
          {
            "in": "body",
            "name": "entry",
            "description": "Todo entry to be added",
            "required": true,
            "schema": {
              "$ref": "#/definitions/TodoEntry"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "OK",
            "schema": {
              "type": "string"
            }
          }
        }
      },
      "delete": {
        "tags": [
          "todo-list-controller"
        ],
        "summary": "Delete all todo entries",
        "description": "Delete all todo entries.",
        "operationId": "deleteTodoEntries",
        "consumes": [
          "application/json"
        ],
        "produces": [
          "*/*"
        ],
        "responses": {
          "200": {
            "description": "OK"
          }
        }
      }
    }
  },
  "definitions": {
    "Attachment": {
      "type": "object",
      "properties": {
        "cid": {
          "type": "string"
        },
        "contentType": {
          "type": "string"
        },
        "data": {
          "type": "string"
        }
      }
    },
    "TodoEntry": {
      "type": "object",
      "properties": {
        "attachment": {
          "$ref": "#/definitions/Attachment"
        },
        "description": {
          "type": "string"
        },
        "done": {
          "type": "boolean"
        },
        "id": {
          "type": "string",
          "format": "uuid"
        },
        "title": {
          "type": "string"
        }
      }
    }
  }
}
```
   
The Swagger API specification defines model objects as well as path resources that a client can call with Http GET, POST, PUT, DELETE and so on.
    
Citrus is able to generate client and server side test cases from that specification. Each path resource is the basis for a new test case. The test will use the information given in the
specification to generate a proper send/receive request/response communication with the REST API. 

The test code generation takes place in the Maven build lifecycle and uses the Citrus Maven plugin. You need to add the following plugin configuration to the Maven POM file:
    
```xml
<plugin>
    <groupId>com.consol.citrus.mvn</groupId>
    <artifactId>citrus-maven-plugin</artifactId>
    <version>${citrus.version}</version>
    <executions>
      <execution>
        <id>generate-tests</id>
        <phase>generate-test-sources</phase>
        <goals>
          <goal>generate-tests</goal>
        </goals>
      </execution>
    </executions>
    <configuration>
      <type>java</type>
      <framework>testng</framework>
      <tests>
        <test>
          <endpoint>todoClient</endpoint>
          <disabled>true</disabled>
          <swagger>
            <file>file://${project.basedir}/src/test/resources/swagger/TodoList.json</file>
          </swagger>
        </test>
      </tests>
    </configuration>
</plugin>
```
        
The plugin test generation binds to the `generate-test-sources` Maven lifecycle phase and generates the tests in a default output directory `target/generated/citrus`. The plugin needs the path to the Swagger Open PI specification file
which is usually a `.json` or `.yaml` file in your project.
 
When executed the plugin generates the Citrus tests to the target output directory. You can add those generated files to the Maven sources/resources with another plugin called `build-helper-maven-plugin`.
        
```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>build-helper-maven-plugin</artifactId>
    <version>3.0.0</version>
    <executions>
      <execution>
        <id>add-test-sources</id>
        <phase>generate-test-sources</phase>
        <goals>
          <goal>add-test-source</goal>
        </goals>
        <configuration>
          <sources>
            <source>${project.build.directory}/generated/citrus/java</source>
          </sources>
        </configuration>
      </execution>
      <execution>
        <id>add-test-resources</id>
        <phase>generate-test-resources</phase>
        <goals>
          <goal>add-test-resource</goal>
        </goals>
        <configuration>
          <resources>
            <resource>
              <directory>${project.build.directory}/generated/citrus/resources</directory>
            </resource>
          </resources>
        </configuration>
      </execution>
    </executions>
</plugin>
```
            
The build helper plugin adds the generated tests to the Maven project source/resource compilation. This automatically enables the generated tests for execution with the Maven build lifecycle in test/integration-test phase.

Run
---------

**NOTE:** This test depends on the [todo-app](../todo-app/) WAR which must have been installed into your local maven repository using `mvn clean install` beforehand.

The sample application uses Maven as build tool. So you can compile, package and test the
sample with Maven.
 
     mvn clean verify -Dembedded
    
This executes the complete Maven build lifecycle. The embedded option automatically starts a Jetty web
container before the integration test phase. The todo-list system under test is automatically deployed in this phase.
After that the Citrus test cases are able to interact with the todo-list application in the integration test phase.

During the build you will see Citrus performing some integration tests.
After the tests are finished the embedded Jetty web container and the todo-list application are automatically stopped.
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#generate
