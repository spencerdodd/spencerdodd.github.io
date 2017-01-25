---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 7"
date:   2017-01-24 19:02:00 -0500
categories: webgoat
---
# Concurrency: Thread Safety Problems
Instructions:
* The user should be able to exploit the concurrency error in this web application and view login information for another user that is attempting the same function at the same time. This will require the use of two browsers. Valid user names are 'jeff' and 'dave'.

Let's take a peek at how the request to the server is formed:

```
username=jeff&SUBMIT=Submit
```

Well, nothing special there going on. So, let's queue up requests for both Jeff and Dave in separate browsers and set Burp up to intercept. Once Burp has caught both requests, we release them both as quickly to simultaneously as possible to maximize any problematic concurrency window. 

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_7/thread-success.jpg">

Burp shows me that the request for Jeff went in first, followed by the request for Dave. Both windows received Dave's information.

Here is some psuedocode of the server side code for this challenge:
```
current_user = None

def user_login(new_user):
	if current_user is None:
		current_user = new_user
		thread.sleep(1.5)
		return current_user.info
	else:
		current_user = new_user
		return current_user.info

```

Server side, what happens is this. The first user logs in and sets the global variable of ```current_user``` to themselves and enters a thread sleep for 1500 ms (1.5 seconds). After the sleep, the user returns the info of the ```current_user```, which it assumes to be them. However, while the first user was sleeping, the second user logged in. The ```user_login``` function then assigned the value of ```current_user``` to the value of the second user, and returned that data to the second user. When the first user wakes up from the thread sleep, it returns the info for the ```current_user``` which it assumes to be itself. However, the second user changed the global variable value to itself while the first user was sleeping, causing them both to return the values for the second user that logged in.

This is why you need to take careful consideration into designing threaded programs with shared resources, and ensure that this sort of concurrency issue doesn't arise.