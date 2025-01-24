---
layout: sample
title: SQL sample
name: sample-jdbc
icon:  database
folder: samples-db
group: endpoints
description: Validates stored data in relational database
categories: [samples]
permalink: /samples/jdbc/
---

This sample uses JDBC database connection to verify stored data in SQL query results sets.

Objectives
---------

The [todo-list](/samples/todo-app/) sample application stores data to a relational database. This sample shows 
the usage of database JDBC validation actions in Citrus. We are able to execute SQL statements on a database target. 
See the [reference guide][1] database chapter for details.

The database source is configured as Spring datasource in the application context ***citrus-context.xml***.
    
```java
@Bean
public SingleConnectionDataSource dataSource() {
    SingleConnectionDataSource dataSource = new SingleConnectionDataSource();
    dataSource.setDriverClassName(JdbcDriver.class.getName());
    dataSource.setUrl("jdbc:citrus:http://localhost:3306/testdb");
    dataSource.setUsername("sa");
    dataSource.setPassword("");
    return dataSource;
}
```
    
As you can see we are using a special Citrus JDBC driver here. This driver connects to the Citrus JDBC server mock.    

In the test case we can verify any JDBC operation on the datasource without having to actually create the data in the database.

```java
receive(jdbcServer)
        .messageType(MessageType.JSON)
        .message(JdbcMessage.execute("SELECT id, title, description FROM todo_entries"));

send(jdbcServer)
        .messageType(MessageType.JSON)
        .message(JdbcMessage.success().dataSet("[ {" +
                    "\"id\": \"" + UUID.randomUUID().toString() + "\"," +
                    "\"title\": \"${todoName}\"," +
                    "\"description\": \"${todoDescription}\"," +
                    "\"done\": \"false\"" +
                "} ]"));
```
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#jdbc
