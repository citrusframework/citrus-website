---
layout: sample
title: TestNG data provider sample
name: sample-dataprovider
group: samples-testng
description: Shows TestNG data provider usage in Citrus
categories: [samples]
permalink: /samples/dataprovider/
---

This sample demonstrates how to use TestNG data providers in Citrus tests. You can also read about this in [reference guide][1].

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides a REST API for managing todo entries.
Citrus is able to call the API methods as a client in order to add new todo entries. In this sample we make use of
the TestNG data provider feature in terms of adding multiple todo entries within on single test.

The data provider is defined in the test case.

```java
@DataProvider(name = "todoDataProvider")
public Object[][] todoDataProvider() {
    return new Object[][] {
        new Object[] { "todo1", "Description: todo1", false },
        new Object[] { "todo2", "Description: todo2", true },
        new Object[] { "todo3", "Description: todo3", false }
    };
}
```
    
The provider gives us two parameters **todoName** and **todoDescription**. The parameters can be bound to test variables
in the Citrus test with some annotation magic.
    
```java
@Test(dataProvider = "todoDataProvider")
@CitrusTest
@CitrusParameters( { "todoName", "todoDescription", "done" })
public void testProvider(String todoName, String todoDescription, boolean done) {
    variable("todoId", "citrus:randomUUID()");

    http()
        .client(todoClient)
        .send()
        .post("/api/todolist")
        .messageType(MessageType.JSON)
        .contentType(ContentType.APPLICATION_JSON.getMimeType())
        .payload("{ \"id\": \"${todoId}\", \"title\": \"${todoName}\", \"description\": \"${todoDescription}\", \"done\": ${done}}");
    
    [...]    
}            
```
        
As you can see we are able to use the name and description values provided by the data provider. When executed the test performs
multiple times with respective values:

```
CITRUS TEST RESULTS
TodoListIT.testPost([todo1, Description: todo1, false]) ............... SUCCESS
TodoListIT.testPost([todo2, Description: todo2, true])  ............... SUCCESS
TodoListIT.testPost([todo3, Description: todo3, false]) ............... SUCCESS    
```    
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/reference/html#run-testng-data-providers