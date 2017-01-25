---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 6"
date:   2017-01-24 19:01:00 -0500
categories: webgoat
---
# Code Quality
Instructions
* Developers are notorious for leaving statements like FIXME's, TODO's, Code Broken, Hack, etc... inside the source code.  Review the source code for any comments denoting  passwords, backdoors, or something doesn't work right.  Below is an example of a forms based authentication form. Look for clues to help you log in. 

Alright let's take a look at the source and search for any of those strings.

```
<!-- FIXME admin:adminpw  -->
<!-- Use Admin to regenerate database  -->
```

Well, okay. Let's give those a shot:

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_6/admin.jpg">

Not a whole lot to that challenge, although it is a good tip. People leave sensitive creds in publicly posted places pretty often. At least often enough that people are scraping Github for AWS keys to run EC2 instances for bitcoin miners. [Source][bitcoin-miners]

[bitcoin-miners]:https://it.slashdot.org/story/15/01/02/2342228/bots-scanning-github-to-steal-amazon-ec2-keys