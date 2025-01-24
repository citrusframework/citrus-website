---
layout: sample
title: SCP sample
name: sample-scp
icon: ftp
folder: samples-ftp
group: endpoints
description: SCP client and server interaction in Citrus
categories: [samples]
permalink: /samples/scp/
---

This sample works with secure copy aka SCP in order to copy files as client to a server provided by Citrus. The scp-client uses the upload for storing a new file on the 
server. After that the very same file is downloaded via SCP in a single test.

Common FTP features are also described in detail in [reference guide][1]

Objectives
---------

We want to setup both SCP client and server components in our project in order to test a file transfer via secure copy. The client authenticates to the server
using username password credentials. The secure ftp-server component will receive incoming requests for validation and provide the user account workspace to the client.

First of all let us setup the necessary components in the Spring bean configuration:

```java
@Bean
public ScpClient scpClient() {
    return CitrusEndpoints.scp()
            .client()
            .port(2222)
            .username("citrus")
            .password("admin")
            .privateKeyPath("classpath:ssh/citrus.priv")
            .build();
}

@Bean
public SftpServer sftpServer() {
    return CitrusEndpoints.sftp()
            .server()
            .port(2222)
            .autoStart(true)
            .user("citrus")
            .password("admin")
            .allowedKeyPath("classpath:ssh/citrus_pub.pem")
            .build();
}
```

The *sftpServer* is a small but fully qualified FTP server implementation that uses secure authentication in Citrus. The server receives an `allowedKeyPath` that defines allowed public keys. Also we define a user and password
for the test user on the server component. The user `citrus` is now able to authenticate with the server via private key. Also the client may authenticate to the server using the given username password credentials. 

Based on the user account we can set a user workspace home directory. The server will save incoming stored files to this directory and the server will read retrieved files from that
home directory.

In case you want to setup some files in that directory in order to provide it to clients, please copy those files to that home directory prior to the test.  

The scp-client connects to the server using the user credentials and is then able to store and retrieve files in a test via SCP.

In our test we can now start to upload a file using SCP.

```java
echo("Store file via SCP");

send(scpClient)
   .fork(true)
   .message(FtpMessage.put("classpath:todo/entry.json", "todo.json", DataType.ASCII));

receive(sftpServer)
    .message(FtpMessage.put("@ignore@", "todo.json", DataType.ASCII));

send(sftpServer)
    .message(FtpMessage.success());

receive(scpClient)
    .message(FtpMessage.success());
```

Now we have both client and server interaction in the same test case. This requires us to use `fork` option enabled on all client
requests as we need to continue with the test in order to handle the server interaction, too. We can store a new file `todo/entry.json` which is transmitted
to the server via secure copy.

The SFTP server is receiving the file upload providing a success response in order to mark completion of the file transfer. After that the file should be created in
the user home directory as file `todo.json`. You can validate the file content by reading it from that directory in another test action.

Lets download that very same file in another SCP file transfer:

```java
echo("Retrieve file from server");

send(scpClient)
    .fork(true)
    .message(FtpMessage.get("todo.json", "file:target/scp/todo.json", DataType.ASCII));

receive(sftpServer)
    .message(FtpMessage.get("/todo.json", "@ignore@", DataType.ASCII));

send(sftpServer)
    .message(FtpMessage.success());

receive(scpClient)
    .message(FtpMessage.success());
```

This completes our test as we were able to interact with the SFTP server using the client secure copy operations.

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

 [1]: https://citrusframework.org/citrus/reference/html#ftp
