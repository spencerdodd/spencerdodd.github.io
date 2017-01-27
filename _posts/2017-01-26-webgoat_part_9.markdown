---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 9"
date:   2017-01-26 13:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Stored XSS Attacks
Instructions:

 - It is always a good practice to scrub all input, especially those inputs that will later be used as parameters to OS commands, scripts, and database queries. It is particularly important for content that will be permanently stored somewhere in the application. Users should not be able to create message content that could cause another user to load an undesireable page or undesireable content when the user's message is retrieved.

 I have my suspicions that this web admin did not take the proper precautions in setting up this message board. Let's try to perform a stored XSS attack by inserting a malicious message into this message board.

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_9/test-message.jpg">

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_9/test-message-logged.jpg">

Alright cool, so it looks like we have stored our payload in the message board. Let's see what happens when a user wants to see our post.

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_9/test-message-clicked.jpg">

SucceXXS!

# Reflected XSS Attacks
Instructions:

- It is always a good practice to validate all input on the server side. XSS can occur when unvalidated user input is used in an HTTP response. In a reflected XSS attack, an attacker can craft a URL with the attack script and post it to another website, email it, or otherwise get a victim to click on it. 

That PIN input looks like a prime target for some sweet sweet reflective XSS action. Let's give it a shot with the industry standard ```<script>alert("XSS");</script>```:

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_9/checkout-insert.jpg">

Generates:

```
192.168.56.101/WebGoat/attack?Screen=31&menu=90&QTY1=1&QTY2=1&QTY3=1&QTY4=1&field2=4128+3214+0002+1999&field1=%3Cscript%3Ealert%28%22XSS%22%29%3B%3C%2Fscript%3E&SUBMIT=Purchase
```

Let's see what happens when we get someone to click on it

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_9/checkout-submit.jpg">

SeXXSy