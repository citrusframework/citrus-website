---
layout: sample
title: Http form data sample
name: sample-http-form-data
icon: globe
folder: samples-http
group: validation
description: Exchange form data via Http GET/POST
categories: [samples]
permalink: /samples/http-form-data/
---

Browser clients are able to post form data as `application/x-www-form-urlencoded` key-value String to the Http server. The HTML forms use `POST` method and build a
form urlencoded String of all form controls and their values as message payload. You can send and receive such form data in Citrus.

Http form data features are also described in detail in [reference guide][1]

Objectives
---------

The test will simulate client and server for form data exchange via Http POST. First of all we need client and server components in the Spring bean configuration:

```java
@Bean
public HttpClient todoClient() {
    return CitrusEndpoints.http()
                        .client()
                        .requestUrl("http://localhost:8080")
                        .build();
}

@Bean
public HttpServer todoListServer() {
    return CitrusEndpoints.http()
            .server()
            .port(8080)
            .timeout(10000)
            .autoStart(true)
            .build();
}
```

These components are normal Citrus Http client and server components. In this test we will use both client and server within the same test case in order to demonstrate both sides
of the communication. In a real world scenario you may just have one side client *or* server in your test. 

So lets start writing a client request that uses form urlencoded message content:

```java
http()
    .client(todoClient)
    .send()
    .post("/api/todo")
    .fork(true)
    .contentType(MediaType.APPLICATION_FORM_URLENCODED_VALUE)
    .payload("title=${todoName}&description=${todoDescription}");
```

As you can see we are using a Http `POST` request with form urlencoded message body. The form data uses two fields `title` and `description` with respective values. On the server side we are able 
to receive this form data for validation:

```java
http()
    .server(todoListServer)
    .receive()
    .post("/api/todo")
    .contentType(MediaType.APPLICATION_FORM_URLENCODED_VALUE)
    .messageType(MessageType.PLAINTEXT)
    .payload("{description=[${todoDescription}], title=[${todoName}]}");

http()
    .server(todoListServer)
    .respond(HttpStatus.OK);
```

The Citrus Http server is automatically handling the form data and converts the data to a list of fields with its values. We can use the plaintext message validation to check that the form data is
as expected. 

In order to use the special `x-www-form-urlencoded` message validation features in Citrus we need to add a new message validator in the configuration.

```java
@Bean
public FormUrlEncodedMessageValidator formUrlEncodedMessageValidator(MessageValidatorRegistry messageValidatorRegistry) {
    FormUrlEncodedMessageValidator messageValidator = new FormUrlEncodedMessageValidator();
    messageValidatorRegistry.getMessageValidators().add(messageValidator);
    return messageValidator;
}
```

The `com.consol.citrus.http.validation.FormUrlEncodedMessageValidator` validator implementation provides convenient form data marshalling that we can use in our test cases when expecting form urlencoded message content.

```java
http()
    .server(todoListServer)
    .receive()
    .post("/api/todo")
    .contentType(MediaType.APPLICATION_FORM_URLENCODED_VALUE)
    .messageType(FormUrlEncodedMessageValidator.MESSAGE_TYPE)
    .payload(getFormData(), new FormMarshaller());

http()
    .server(todoListServer)
    .respond(HttpStatus.OK);
```

As you can see we can now use the special message type `x-www-form-urlencoded` which enables the `FormUrlEncodedMessageValidator` mechanism. This allows us to specify the full form data as plain Java object.

```java
private FormData getFormData() {
    FormData formData = new FormData();

    formData.setAction("/api/todo");
    formData.setContentType(MediaType.APPLICATION_FORM_URLENCODED_VALUE);

    Control description = new Control();
    description.setName("description");
    description.setValue("@ignore@");
    formData.addControl(description);

    Control title = new Control();
    title.setName("title");
    title.setValue("${todoName}");
    formData.addControl(title);

    return formData;
}
```

This is how to send and receive Http form data in Citrus.
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#http-rest
