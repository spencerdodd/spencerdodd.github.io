---
layout: post
title:  "Root-Me App Script 2"
subtitle: "Python - input()"
date:   2017-02-08 02:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-net-header.jpg"
---

{% highlight python %}
#!/usr/bin/python2
 
import sys
 
def youLose():
    print "Try again ;-)"
    sys.exit(1)
 
 
try:
    p = input("Please enter password : ")
except:
    youLose()
 
 
with open(".passwd") as f:
    passwd = f.readline().strip()
    try:
        if (p == int(passwd)):
            print "Well done ! You can validate with this password !"
    except:
        youLose()
{% endhighlight %}

