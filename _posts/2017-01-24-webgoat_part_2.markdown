---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 2"
date:   2017-01-24 00:55:00 -0500
categories: webgoat
---
# Access Control Flaws Part 1: Bypassing Path-based Authentication
Whew, this looks like it has a little less research involved that the HTTP Response Splitting attack. We are given a window with a list of files that belong to the directory ```/var/lib/tomcat6/webapps/WebGoat/lesson_plans/English```. We are told that our current login, 'user', is given access to the ```lesson_plans/English``` directory. Our goal is to access files that are not in our granted directory.

<img src= "{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/part2window.jpg">

So it appears that our access is limited mainly by selectability in the window. However, let's take a look at the request that is sent when we select an image from the window:

<img src="{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/part2request.jpg">

So it looks like our request simply has the file name, and the actual accession of the file done by the server prepends the authenticated path to the given file name. Fortunately for us, there is an in-path modifier for parent/child directory traversal: ```..``` Using this trick, let's see if we can access things outside of the directory we are dropped into in the window.

First I took a look at the directory structure in the VM. I'm not sure if that is allowed, but I couldn't get the tomcat link to work (I've found a few discrepancies between intended and actual behaviour with some paths and lessons on this version of WebGoat). After searching around I found that there is a German directory in our ```.../lesson_plans/``` directory. So I craft up another request that takes use of the ```..``` command for parent of the working directory:

<img src=" {{ site.baseurl }}/2017-01-24-webgoat_part_2/images/part2requestattempt.jpg">

And bingo! We have access.

The lesson here is that controlling access by simply limiting functional view client-side does not prevent request-tampering by savvy users or attackers.

# Access Control Flaws Part 2: Bypassing Presentation Level Access Control
For this challenge we are givin a login screen for an employee portal at Goat Hills Financial. Our goal is, as regular user 'Tom', to exploit weak access control to use the admin-only```Delete``` function to delete a user profile. We are givin all the login information for the users (password=firstname in lowercase), so lets take a look at what a legitimate admin usage of ```Delete``` looks like:

First, we login as admin user "John":

```
POST /WebGoat/attack?Screen=65&menu=200 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=65&menu=200&stage=1
Cookie: JSESSIONID=D693B413D068E6FBF713D72CD2522A3B; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 42

employee_id=111&password=john&action=Login
```

We are presented with the admin control page.

<img src="{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/admin-page.jpg">

Next, we delete Tom's profile:

```
POST /WebGoat/attack?Screen=65&menu=200 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=65&menu=200
Cookie: JSESSIONID=D693B413D068E6FBF713D72CD2522A3B; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 36

employee_id=105&action=DeleteProfile
```
Hmm. It looks like there is no verification that the user requesting the deletion is an admin in the request. While the button linked to the ```Delete``` function is only accessible from the admin control panel, we can craft our own request while we are not logged in as an admin and it should function identically as if we were.

So we log back into Tom's account and press the ```ViewProfile``` button. Intercepting the request, we change the action to DeleteProfile:

<img src="{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/tom-delete.jpg">

Got 'em.

# Access Control Flaws Part 3: Bypassing Data Level Access Control
So for this part of the challenge, our goal is to view another user's profile. To start, lets log back into Tom's account and press ```ViewProfile```. Intercepting that request, we can see that Tom's ```employee_id``` is 105. 

```
POST /WebGoat/attack?Screen=65&menu=200 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=65&menu=200
Cookie: JSESSIONID=D693B413D068E6FBF713D72CD2522A3B; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 34

employee_id=105&action=ViewProfile
```
Assuming these are sequentially assigned, why don't we just try to change it to another close number that should be in our user set? So we alter the ```employee_id``` field to 103 to try to take a look at Curly's profile, and:

<img src="{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/tom-view.jpg">

Boom. Curly's paid/owed ratio isn't too hot.

# Access Control Flaws Final: Remote Admin Access
The goal for this final section is to access the admin interface for WebGoat. One of the given hints is that the admin interface is hackable via URL. After messing around for a while (too long to admit), the idea of URL-field specifications came to mind. I tried something super simple, and appended ```&admin=true``` onto the end of the URL...

<img src="{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/admin-functions.jpg">

Well, alright then. So we get access to this whole set of admin functions. Trying ```User Information```, we intercept the request and add ```&admin=true``` onto it (to ensure we keep our admin status). This yields us a list of users:

<img src="{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/admin-function-users.jpg">

While ```Product Information``` (again appending the admin tag) yields us this list:

<img src="{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/admin-function-products.jpg">

Clicking ```Adhoc Query``` leads us to a page where we are supposed to enter a SQL statement to post to the message board

<img src="{{ site.baseurl }}/2017-01-24-webgoat_part_2/images/admin-function-sql.jpg">

However, unfortunately the functionality is broken in my version of WebGoat. Maybe I will try to get a newer version running and come back at some point to replace any broken parts of 5.4 with the (hopefully) patched challenges.