---
layout: sample
title: XHTML sample
name: sample-xhtml
icon: code
folder: samples-xml
group: validation
description: Shows XHTML validation feature
categories: [samples]
permalink: /samples/xhtml/
---

This sample uses XHTML validation features to verify HTML response from a web container. This feature is
also described in detail in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application provides an index HTML page that displays all todo entries.
We can validate the HTML content in Citrus with XPath expressions. Citrus automatically converts the HTML to
XHTML so XPath expressions can be evaluated accordingly.

The sample tests show how to use this feature. First we define a global namespace for XHTML in
configuration.

```java
@Bean
public NamespaceContextBuilder namespaceContextBuilder() {
    NamespaceContextBuilder namespaceContextBuilder = new NamespaceContextBuilder();
    namespaceContextBuilder.setNamespaceMappings(Collections.singletonMap("xh", "http://www.w3.org/1999/xhtml"));
    return namespaceContextBuilder;
}
```
    
Now we can use the XHTML validation feature in the Citrus test.
    
```java
http()
    .client(todoClient)
    .response(HttpStatus.OK)
    .messageType(MessageType.XHTML)
    .xpath("(//xh:li[@class='list-group-item']/xh:span)[last()]", "${todoName}");
```
        
In a Http client response we can set the message type to XHTML. Citrus automatically converts the HTML response to
XHTML so we can use XPath to validation the HTML content.

The XPath expression makes sure the the last todo entry displayed is the todo item that we have added before in the test.    
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#validation-xhtml
