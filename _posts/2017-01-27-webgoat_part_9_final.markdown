---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 9 (Final)"
date:   2017-01-27 13:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
Instructions:

- Tomcat is configured to support the HTTP TRACE command. Your goal is to perform a Cross Site Tracing (XST) attack. 

Alright, so what does the TRACE command do? Apparently, it invokes a remote, application-layer loop-back of the request message. The client can see what is being received at the other end of the request chain and use that data for diagnostic information. [w3 Trace Protocol][w3-trace]

Hmm..request loop-back. So we can view the original full request including headers? Well if we could get a user to run a ```TRACE``` command that would mean we could steal session cookies (as long as we can view the ```TRACE``` response). Let's craft a little XSS payload to perform a trace.

```
<script>
function callHTTPTrace() {
	xmlHTTP = new XMLHttpRequest();
	xmlHTTP.open("TRACE", "attack?Screen=75&menu=900", false);
	xmlHTTP.onreadystatechange = function () {
		if (xmlHTTP.readyState == 4 && xmlHTTP.status == 200) {
			alert(xmlHTTP.response)
		}
	}
	xmlHTTP.send();
}
callHTTPTrace();
</script>
```

This code makes an asynchronous ```TRACE``` call to the page we are on using an ```XMLHttpRequest``` object. By reading the response, we should be able to steal cookies from the user who runs this script. When we try to run this XSS though, the browser actually fails this script with the console log:

```
SecurityError: The operation is insecure.
```

However, we pass the lesson (nice?). If we check the challenge source, we can see the code that decides the outcome of our attempt. 

```
if (param1.toLowerCase().indexOf("script") != -1 && param1.toLowerCase().indexOf("trace") != -1)
{
	makeSuccess(s);
}
```

Aha, so we passed the lesson by including the word 'trace' in our input even though our script itself was blocked by the browser. Not the most comprehensive functional check. Although this attack doesn't seem to be too effective, at least when implemented so bluntly (like here) as our browser itself blocks the calls.

So on an insecure/outdated browser, this attack could likely still be used. Additionally, only if the ```TRACE``` command is enabled by the server (which many servers don't allow). I'm not sure entirely how useful this attack is, but it could be used to bypass the ```HTTPOnly``` cookie attribute, which disallows reading of cookies that have this flag set.

### Resources

[1. Trace HTTP Command][w3-trace]
[2. HTTPOnly Cookie Attribute][http-only]

[w3-trace]:https://www.w3.org/Protocols/rfc2616-sec9.html
[http-only]:https://www.owasp.org/index.php/HTTPOnly#Browsers_Supporting_HTTPOnly