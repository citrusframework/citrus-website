---
layout: sample
title: SQL sample
name: sample-sql
group: samples-db
description: Execute SQL statements in Citrus
categories: [samples]
permalink: /samples/sql/
---

This sample uses JDBC database connection to verify stored data in SQL query results sets.

Objectives
---------

The [todo-list](../todo-app/README.md) sample application stores data to a relational database. This sample shows 
the usage of database JDBC validation actions in Citrus. We are able to execute SQL statements on a database target. 
See the [reference guide][1] database chapter for details.

The database source is configured as Spring datasource in the application context ***citrus-context.xml***.
    
```java
@Bean(destroyMethod = "close")
public BasicDataSource todoListDataSource() {
    BasicDataSource dataSource = new BasicDataSource();
    dataSource.setDriverClassName("org.hsqldb.jdbcDriver");
    dataSource.setUrl("jdbc:hsqldb:hsql://localhost/testdb");
    dataSource.setUsername("sa");
    dataSource.setPassword("");
    dataSource.setInitialSize(1);
    dataSource.setMaxActive(5);
    dataSource.setMaxIdle(2);
    return dataSource;
}
```
    
As you can see we are using a H2 in memory database here.    

Before the test suite is started we create the relational database tables required.

```java
@Bean
public SequenceBeforeSuite beforeSuite() {
    return new TestDesignerBeforeSuiteSupport() {
        @Override
        public void beforeSuite(TestDesigner designer) {
            designer.sql(todoListDataSource())
                .statement("CREATE TABLE IF NOT EXISTS todo_entries (id VARCHAR(50), title VARCHAR(255), description VARCHAR(255), done BOOLEAN)");
        }
    };
}
```

After the test we delete all test data again.

```java
@Bean
public SequenceAfterSuite afterSuite() {
    return new TestDesignerAfterSuiteSupport() {
        @Override
        public void afterSuite(TestDesigner designer) {
            designer.sql(todoListDataSource())
                .statement("DELETE FROM todo_entries");
        }
    };
}
```

In the test case we can reference the datasource in order to access the stored data and
verify the result sets.

```java
query(todoDataSource)
    .statement("select count(*) as cnt from todo_entries where title = '${todoName}'")
    .validate("cnt", "1");
```

Run
---------

You can execute some sample Citrus test cases in this sample in order to write the reports.
Open a separate command line terminal and navigate to the sample folder.

Execute all Citrus tests by calling

     mvn integration-test

You should see Citrus performing several tests with lots of debugging output. 
And of course green tests at the very end of the build and some new reporting files in `target/citrus-reports` folder.

Of course you can also start the Citrus tests from your favorite IDE.
Just start the Citrus test using the TestNG IDE integration in IntelliJ, Eclipse or Netbeans.

 [1]: https://citrusframework.org/reference/html#actions-database