---
layout: post
title:  "OWASP BWA WebGoat Challenge: Injection Flaws"
subtitle: "Log Spoofing"
date:   2017-01-30 01:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Log Spoofing
Instructions:

- The grey area below represents what is going to be logged in the web server's log file.
- Your goal is to make it like a username "admin" has succeeded into logging in.
- Elevate your attack by adding a script to the log file. 

Alright, let's see what happens to the log when we throw some test failure input at our login prompt:

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_10_continued/test-login.jpg">

Cool. So, maybe we can add a new-line character followed by what looks like a successful admin login. Let's try this in the username field and see what happens:

```
test%0aLogin succeeded for username: admin
```

Alright! Looks like we've successfully spoofed an admin login. Let's try to inject a little XSS into our input as well:

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_10_continued/test-admin.jpg">

```
test%0aLogin succeeded for username: <script>alert('xss')</script>admin
```

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_10_continued/test-admin-alert.jpg">

Alright! Now when someone looks at the login logs, they will be hit with our XSS attack! This could be especially damaging for the webserver, as the people most likely to be checking logs are admins themselves. Any CSRF attacks using admin level privileges would allow us to do pretty much whatever we wanted to the site and its users.