---
layout: sample
title: Selenium sample
name: sample-selenium
group: none
description: Use Selenium for UI testing
categories: [samples]
permalink: /samples/selenium/
---

This sample shows how to use Selenium in order to create UI testing with Citrus. Why would somebody combine Citrus with Selenium?
On the one hand Citrus provides XML and Java fluent API integration for Selenium tasks so you do not need to write Selenium code yourself. 
Secondly you might want to combine UI testing with Citrus messaging capabilities such as deleting all entries on a Http REST service first and then
creating a new entry via UI interaction in a browser. Read about this feature in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides a REST API for managing todo entries.
We open the browser and perform some user interaction with the todo app user interface. Besides that we call the Http REST API in order to clean up
all todo entries up front. The Selenium integration is also used to validate HTML elements and page objects in our test cases. 

First of all we add a Selenium browser component as Spring bean.

```java
@Bean
public SeleniumBrowser browser() {
    return CitrusEndpoints.selenium()
            .browser()
            .type(BrowserType.CHROME)
            .build();
}
```

As you can see the browser component in Citrus is a normal Selenium WebDriver. You can pick the target browser implementation and 
add web driver configuration as you like.

In addition to that basic browser component we add a normal Http client component for calling the Http REST API on the todo server.

```java
@Bean
public HttpClient todoClient() {
    return CitrusEndpoints.http()
                        .client()
                        .requestUrl("http://localhost:8080")
                        .build();
}
```

Also we can use a new after suite hook in order to close the browser when all tests are finished:

```java
@Bean
@DependsOn("browser")
public SequenceAfterSuite afterSuite(SeleniumBrowser browser) {
    return new TestRunnerAfterSuiteSupport() {
        @Override
        public void afterSuite(TestRunner runner) {
            runner.selenium(builder -> builder.browser(browser).stop());
        }
    };
}
```

For demonstration purpose we add a `sleep` action in a new after test hook so we see actually what the browser is doing.

```java
@Bean
public SequenceAfterTest afterTest() {
    return new TestRunnerAfterTestSupport() {
        @Override
        public void afterTest(TestRunner runner) {
            runner.sleep(500);
        }
    };
}
```

In a test case we can now use the Selenium browser component in the Citrus fluent API.

```java
@Test
@CitrusTest
public void testAddEntry() {
    variable("todoName", "todo_citrus:randomNumber(4)");
    variable("todoDescription", "Description: ${todoName}");

    http()
        .client(todoClient)
        .send()
        .delete("/api/todolist");

    http()
        .client(todoClient)
        .receive()
        .response(HttpStatus.OK);

    selenium()
            .browser(browser)
            .start();

    selenium()
            .navigate(todoClient.getEndpointConfiguration().getRequestUrl() + "/todolist");

    selenium()
            .find()
            .element(By.xpath("(//li[@class='list-group-item'])[last()]"))
            .text("No todos found");

    selenium()
            .setInput("${todoName}")
            .element(By.name("title"));

    selenium()
            .setInput("${todoDescription}")
            .element(By.name("description"));

    selenium().click()
            .element(By.tagName("button"));

    selenium()
            .waitUntil()
            .element(By.xpath("(//li[@class='list-group-item']/span)[last()]"))
            .timeout(2000L)
            .visible();

    selenium()
            .find()
            .element(By.xpath("(//li[@class='list-group-item']/span)[last()]"))
            .text("${todoName}");
}
```

We call the `DELETE` option on the `/todolist` Http resource in order to have an empty todo list that we can operate on. The Selenium
browser is started and the user navigates to the todo list application. The testcase validates the empty todo list display and fills out the todo HTML form
with a sample entry. The form is submitted and a final validation checks that the todo entry is added and displayed in the list.

Page objects
---------

Using page objects that represent a application page with its elements and action capabilities is a good practice in UI testing with Selenium.
Citrus also supports the usage Java POJOs as page objects.

```java
public class TodoPage implements WebPage, PageValidator<TodoPage> {

    @FindBy(tagName = "h1")
    private WebElement heading;

    @FindBy(tagName = "form")
    private WebElement newTodoForm;

    @FindBy(xpath = "(//li[@class='list-group-item'])[last()]")
    private WebElement lastEntry;

    /**
     * Submits new entry.
     * @param title
     * @param description
     */
    public void submit(String title, String description) {
        newTodoForm.findElement(By.id("title")).sendKeys(title);

        if (StringUtils.hasText(description)) {
            newTodoForm.findElement(By.id("description")).sendKeys(description);
        }

        newTodoForm.submit();
    }

    @Override
    public void validate(TodoPage webPage, SeleniumBrowser browser, TestContext context) {
        Assert.assertEquals(heading.getText(), "TODO list");
        Assert.assertEquals(lastEntry.getText(), "No todos found");
    }
}
```

The page object listed above represents the todo list page with its elements and actions such as `submit`. The page also implements the
`PageValidator` interface adding validation methods for checking that the rendered page is as expected.

In a test case we can use this page objects for user interaction and validation:

```java
@Test
@CitrusTest
public void testAddEntry() {
    variable("todoName", "todo_citrus:randomNumber(4)");
    variable("todoDescription", "Description: ${todoName}");

    [...]

    TodoPage todoPage = new TodoPage();

    selenium()
        .page(todoPage)
        .validate();

    selenium()
        .page(todoPage)
        .argument("${todoName}")
        .argument("${todoDescription}")
        .execute("submit");

    [...]
}
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

 [1]: https://citrusframework.org/citrus/reference/html#selenium
