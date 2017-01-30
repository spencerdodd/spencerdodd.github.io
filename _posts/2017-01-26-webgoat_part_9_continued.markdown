---
layout: post
title:  "OWASP BWA WebGoat Challenge: Cross Site Scripting"
subtitle: "Cross-Site Request Forgery (CSRF)"
date:   2017-01-26 20:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Cross-Site Request Forgery (CSRF)
Instructions:

- Your goal is to send an email to a newsgroup that contains an image whose URL is pointing to a malicious request. Try to include a 1x1 pixel image that includes a URL. The URL should point to the CSRF lesson with an extra parameter "transferFunds=4000".

CSRF, Cross-site request forgery, also referred to as a ‘one-click attack’ or ‘session riding’ is a malicious exploit of a website in which unauthorized commands are sent from a user of a website that the victim trusts. The fundamental difference between CSRF and XSS is that cross-site scripting (XSS), is designed to exploit the trust the user has for a particular site whilst CSRF aims to exploit the trust that a website has in the visitor’s browser. So, the key difference is within the victims browser. ([Source][csrf-xss])

After doing some reading, it seems to me that CSRF is most similar to a reflected XSS attack. Both require the user to interact with a malicious link. However, with reflected XSS you can cause the user to execute arbitrary code that you inject into the link. With CSRF, you can perform functions that are accessible to the user on the web application by using their authentication on that application / site.

For example:

Your target is a chat forum. This is a good target because you know that the users need to be authenticated in order to interact on the forum. A normal user function is to change their email. The request that is generated when a user changes their email is:

```
POST http://chat-forum.com/change-email.php?new_email=users_new_email HTTP/1.1
```

So, you create a link that looks like this:

```
POST http://chat-forum.com/change-email.php?new_email=attackers_email HTTP/1.1
```

Then you make a post on the forum and link your maliciously crafted link in the body. Any user that clicks the link will have their email changed to your email, giving you control of their account (after a ```Forgot Password``` reset). While we cannot execute arbitrary code (which gives us more possibilities and leverage in attack), we can abuse existing server functions that affect our target as the server trusts the user's validity even though we were the ones that crafted the request.

So let's look at applying this to our challenge. We need to craft an email that will contain an invisible image (well almost) that includes a modified URL to the CSRF challenge that will perform our attack. The challenge URL is as follows:

```
http://192.168.56.101//WebGoat/attack?Screen=52&menu=900
```

And when we append our payload:

```
http://192.168.56.101//WebGoat/attack?Screen=52&menu=900&transferFunds=4000
```

Now lets make a single pixel image that will trigger this URL:

```
<img src="
http://192.168.56.101//WebGoat/attack?Screen=52&menu=900&transferFunds=4000" alt="transferPixel" style="width:1;height:1;"> 
```

We use this to create a message:

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_9_continued/csrf-message.jpg">

And then it's inserted to the message board. Waiting for any takers. 

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_9_continued/csrf-message-click.jpg">

And whoever clicks it is out $4000!

Bummer.


### Resources

[1. Consise Courses XSS vs. CSRF][csrf-xss]

[csrf-xss]:https://www.concise-courses.com/security/xss-csrf-html5/