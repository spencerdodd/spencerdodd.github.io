---
layout: post
title:  "DVWA Brute Force"
subtitle: "Damn Vulnerable Web Application"
date:   2017-02-08 12:00:00 -0500
author: "coastal"
header-img: "images/site-resources/dvwa-header.jpg"
---

```
GET /dvwa/vulnerabilities/brute/?username=user&password=user&Login=Login HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:51.0) Gecko/20100101 Firefox/51.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/dvwa/vulnerabilities/brute/?username=test&password=test&Login=Login
Cookie: security=low; JSESSIONID=C7D25AE3371371E9E3003D67F7FC0CB0; acopendivids=swingset,dvwa,jotto,phpbb2,redmine; acgroupswithpersist=nada; PHPSESSID=4c0ndf5r1fgerkl58uo6kna4b1
Connection: close
Upgrade-Insecure-Requests: 1
```

Welcome to the password protected area user

```
GET /dvwa/vulnerabilities/brute/?username=test&password=test&Login=Login HTTP/1.1
```

Username and/or password incorrect.

