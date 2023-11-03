---
layout: sample
title: Mail sample
name: sample-mail
group: none
description: Shows mail server activities in Citrus
categories: [samples]
permalink: /samples/mail/
---

This sample demonstrates the usage of mail server activities in Citrus tests. You can also read about this in [reference guide][1].

Objectives
---------

The [todo-list](/samples/todo-app/) sample application sends out mail reports to users on demand.
Citrus is able to trigger the report via Http REST API. In this sample we send out some Http REST calls and
wait for the incoming mail in a single test.

First we need the mail server component in Citrus. Lets add this to the configuration:

```java
@Bean
public MailServer mailServer() {
    return CitrusEndpoints.mail()
            .server()
            .port(2222)
            .autoAccept(true)
            .autoStart(true)
            .build();
}
```
                
Now we can receive the mail in the test case.
    
```java
receive(mailServer)
    .message(MailMessage.request()
                .from("todo-report@example.org")
                .to("users@example.org")
                .cc("")
                .bcc("")
                .subject("ToDo report")
                .body("There are '1' todo entries!", "text/plain; charset=us-ascii"))
    .header(CitrusMailMessageHeaders.MAIL_SUBJECT, "ToDo report");

send(mailServer)
    .message(MailMessage.response(250, "OK"));            
```
        
The mail content is marshalled to an expected XML representation that we expect to arrive in the test.

```xml
<mail-message xmlns="http://www.citrusframework.org/schema/mail/message">
  <from>todo-report@example.org</from>
  <to>users@example.org</to>
  <cc></cc>
  <bcc></bcc>
  <subject>ToDo report</subject>
  <body>
    <contentType>text/plain; charset=us-ascii</contentType>
    <content>There are '${entryCount}' todo entries!</content>
  </body>
</mail-message>
```
        
Citrus is able to convert mail messages to an internal XML representation. This way the content is more comfortable to
compare in validation. The mail response looks like this.

```xml
<mail-response xmlns="http://www.citrusframework.org/schema/mail/message">
  <code>250</code>
  <message>OK</message>
</mail-response>
```
    
In the sample the success code **250** is returned to the mail client marking that everything is ok. Here we also could place
some other code and message in order to simulate mail server problems.    
                
Run
---------

You can run the sample on your localhost in order to see Citrus in action. Read the instructions [how to run](/samples/run/) the sample.

 [1]: https://citrusframework.org/citrus/reference/html#mail
