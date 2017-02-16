---
layout: post
title:  "OWASP BWA WebGoat Challenge: Session Management Flaws"
subtitle: "Session Fixation"
date:   2017-02-01 10:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Session Fixation
Instructions:

- You are Hacker Joe and you want to steal the session from Jane. Send a prepared email to the victim which looks like an official email from the bank. A template message is prepared below, you will need to add a Session ID (SID) in the link inside the email. Alter the link to include a SID

<img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_20/init-message.jpg">

Alright, looks like a nice phishing email we have typed up here. Let's take a look at what 'Session Fixation' actually is. According to OWASP's article, session fixation differs from the previous hijacking attack because as opposed to stealing another valid user's session, we get them to authenticate with a pre-authenticated session that we control. One way we can do that is with a session token in a URL argument.

Okay, so our first step is to insert a session ID into the email:

```
<b>Dear MS. Plane</b> <br><br>During the last week we had a few problems with our database. We have received many complaints regarding incorrect account details. Please use the following link to verify your account data:<br><br><center><a href=/webgoat/attack?Screen=284&menu=1800> Goat Hills Financial</a></center><br><br>We are sorry for the any inconvenience and thank you for your cooparation.<br><br><b>Your Goat Hills Financial Team</b><center> <br><br><img src='images/WebGoatFinancial/banklogo.jpg'></center>
```

Let's throw ```&SID=1``` onto the webgoat URL in the email (and change ```webgoat``` to ```WebGoat``` due to the broken links)

<img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_20/sid-insert-success.jpg">

And when we click the link:

<img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_20/sid-link-click.jpg">

Alright, so now we are given the instructions to the next stage

- The bank has asked you to verfy your data. Log in to see if your details are correct. Your user name is Jane and your password is tarzan. 

<img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_20/jane-login.jpg">

Awesome, Jane has authenticated. Now let's go steal her info. We send off a request with the now authenticated pre-fixed session:

```
GET /WebGoat/attack?Screen=284&menu=1800&SID=001 HTTP/1.1
```

<img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_20/jane-steal.jpg">

And we've stolen her account!

### References

[1. OWASP Session Fixation][owasp-fixation]

[owasp-fixation]:https://www.owasp.org/index.php/Session_fixation