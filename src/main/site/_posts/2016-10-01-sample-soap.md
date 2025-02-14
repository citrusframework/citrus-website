---
layout: sample
title: SOAP WS sample
name: sample-soap
icon:  soap
folder: samples-soap
group: endpoints
description: Shows SOAP web service support
categories: [samples]
permalink: /samples/soap/
---

This sample uses SOAP web services to add new todo entries on the todo app system under test. You can read more about the 
Citrus SOAP features in [reference guide][1]

Objectives
---------

The [todo-list](/samples/todo-app/) sample application manages todo entries. The application provides a SOAP web service
endpoint for adding new entries and listing all entries.

The sample tests show how to use this SOAP endpoint as a client. First we define the schema and a global namespace for the SOAP
messages.

```java
@Bean
public SimpleXsdSchema todoListSchema() {
    return new SimpleXsdSchema(new ClassPathResource("schema/TodoList.xsd"));
}

@Bean
public XsdSchemaRepository schemaRepository() {
    XsdSchemaRepository schemaRepository = new XsdSchemaRepository();
    schemaRepository.getSchemas().add(todoListSchema());
    return schemaRepository;
}

@Bean
public NamespaceContextBuilder namespaceContextBuilder() {
    NamespaceContextBuilder namespaceContextBuilder = new NamespaceContextBuilder();
    namespaceContextBuilder.setNamespaceMappings(Collections.singletonMap("todo", "http://citrusframework.org/samples/todolist"));
    return namespaceContextBuilder;
}
```
   
The schema repository holds all known schemas in this project. Citrus will automatically check the syntax rules for incoming messages
then. Next we need a SOAP web service client component:

```java
@Bean
public SoapMessageFactory messageFactory() {
    return new SaajSoapMessageFactory();
}

@Bean
public WebServiceClient todoClient() {
    return CitrusEndpoints.soap()
                        .client()
                        .defaultUri("http://localhost:8080/services/ws/todolist")
                        .build();
}
```
    
The client connects to the web service endpoint on the system under test. In addition to that we define a SOAP message factory that is
responsible for creating the SOAP envelope. 

Now we can use the web service client in the Citrus test with SOAP request and attachment.
    
```java
soap()
    .client(todoClient)
    .send()
    .soapAction("addTodoEntry")
    .payload(new ClassPathResource("templates/addTodoEntryRequest.xml"))
    .attachment("myAttachment", "text/plain", "This is my attachment");
    
soap()
    .client(todoClient)
    .receive()
    .payload(new ClassPathResource("templates/addTodoEntryResponse.xml"));
```
        
The Citrus test sends a request with attachment data. The attachment is transmitted as text data via Http to the server. 
The todo-list WebService endpoint will recognize the attamchent data and add it to the todo entry. So we can expect the attachment data to be returned in
the list of todo entries.
        
```java
soap()
    .client(todoClient)
    .receive()
    .payload(new ClassPathResource("templates/addTodoEntryResponse.xml"));

soap()
    .client(todoClient)
    .send()
    .soapAction("getTodoList")
    .payload(new ClassPathResource("templates/getTodoListRequest.xml"));
```
            
And in the expected message payload we validate the attachment data returned by the server.
            
```xml
<todo:getTodoListResponse xmlns:todo="http://citrusframework.org/samples/todolist">
  <todo:list>
    <todo:todoEntry>
      <todo:id>@ignore@</todo:id>
      <todo:title>${todoName}</todo:title>
      <todo:description>${todoDescription}</todo:description>
      <todo:attachment>
        <todo:cid>myAttachment</todo:cid>
        <todo:contentType>text/plain</todo:contentType>
        <todo:data>citrus:encodeBase64('This is my attachment')</todo:data>
      </todo:attachment>
    </todo:todoEntry>
  </todo:list>
</todo:getTodoListResponse>
```    
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#soap
