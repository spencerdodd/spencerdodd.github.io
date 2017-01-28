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

Hmm..request loop-back. So we can view the full request including headers? Well if we could get a user to run a ```TRACE``` command that would mean we could steal session cookies (as long as we can view the ```TRACE``` response). Let's craft a little XSS payload to perform a trace.

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

The browser actually fails this script with the console log:

```
SecurityError: The operation is insecure.
```

However, we pass the lesson. If we check the source, we can see that it is 

```
if (param1.toLowerCase().indexOf("script") != -1 && param1.toLowerCase().indexOf("trace") != -1)
{
	makeSuccess(s);
}
```

So on an insecure/outdated browser, this attack could likely still be used. Additionally, only if the ```TRACE``` command is enabled by the server (which many servers don't allow). I'm not sure entirely how useful this attack is when it is deployed like this.

### Resources

[w3-trace]:https://www.w3.org/Protocols/rfc2616-sec9.html