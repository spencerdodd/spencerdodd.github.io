---
layout: post
title:  "OWASP BWA WebGoat .NET Challenge"
subtitle: "Web Proxy Test"
date:   2017-02-04 01:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-net-header.jpg"
---
Instructions:

- Set up your web proxy, and test it here. The field has a validator that only allows valid characters. See if you can circumvent this protection using your proxy! 

<img src="{{ site.baseurl }}/images/webgoat-net/2017-02-04-webgoat-net_part_1/web-proxy.jpg">

```
Server=b3dhc3Bid2E=;
```

```
>>> base64.b64decode('b3dhc3Bid2E=')
'owaspbwa'
```

```
__VIEWSTATE=DAwADgIBG2N0bDAwJEhlYWRMb2dpblN0YXR1cyRjdGwwMQEbY3RsMDAkSGVhZExvZ2luU3RhdHVzJGN0bDAzAAA%3D&ctl00%24BodyContentPlaceholder%24txtName=test&ctl00%24BodyContentPlaceholder%24btnReverse=Submit&__EVENTVALIDATION=GwABAAAA%2F%2F%2F%2F%2FwEAAAAAAAAADwEAAAAEAAAACAZFC0eLh7q7fJ1DpeXA3eQLAA%3D%3D&__EVENTTARGET=&__EVENTARGUMENT=
```

```
__VIEWSTATE=DAwADgIBG2N0bDAwJEhlYWRMb2dpblN0YXR1cyRjdGwwMQEbY3RsMDAkSGVhZExvZ2luU3RhdHVzJGN0bDAzAAA=&ctl00$BodyContentPlaceholder$txtName=test&ctl00$BodyContentPlaceholder$btnReverse=Submit&__EVENTVALIDATION=GwABAAAA/////wEAAAAAAAAADwEAAAAEAAAACAZFC0eLh7q7fJ1DpeXA3eQLAA==&__EVENTTARGET=&__EVENTARGUMENT=
```

__VIEWSTATE

```
base64.b64decode('DAwADgIBG2N0bDAwJEhlYWRMb2dpblN0YXR1cyRjdGwwMQEbY3RsMDAkSGVhZExvZ2luU3RhdHVzJGN0bDAzAAA=')
'\x0c\x0c\x00\x0e\x02\x01\x1bctl00$HeadLoginStatus$ctl01\x01\x1bctl00$HeadLoginStatus$ctl03\x00\x00'
```
