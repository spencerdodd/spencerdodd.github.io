---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 2"
date:   2017-01-24 00:50:00 -0500
categories: coding security owasp bwa
---
# Access Control Flaws Part 1: Bypassing Path-based Authentication
Whew, this looks like it has a little less research involved that the HTTP Response Splitting attack. We are given a window with a list of files that belong to the directory ```/var/lib/tomcat6/webapps/WebGoat/lesson_plans/English```. We are told that our current login, 'user', is given access to the ```lesson_plans/English``` directory. Our goal is to access files that are not in our granted directory.

<img src= "{{ site.baseurl }}/images/part2window.jpg">

So it appears that our access is limited mainly by selectability in the window. However, let's take a look at the request that is sent when we select an image from the window:

<img src="{{ site.baseurl }}/images/part2request.jpg">

So it looks like our request simply has the file name, and the actual accession of the file done by the server prepends the authenticated path to the given file name. Fortunately for us, there is an in-path modifier for parent/child directory traversal: ```..``` Using this trick, let's see if we can access things outside of the directory we are dropped into in the window.

First I took a look at the directory structure in the VM. I'm not sure if that is allowed, but I couldn't get the tomcat link to work (I've found a few discrepancies between intended and actual behaviour with some paths and lessons on this version of WebGoat). After searching around I found that there is a German directory in our ```.../lesson_plans/``` directory. So I craft up another request that takes use of the ```..``` command for parent of the working directory:

<img src=" {{ site.baseurl }}/images/part2requestattempt.jpg">

And bingo! We have access.

The lesson here is that controlling access by simply limiting functional view client-side does not prevent request-tampering by savvy users or attackers.