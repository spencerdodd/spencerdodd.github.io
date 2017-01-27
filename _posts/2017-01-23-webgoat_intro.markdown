---
layout: post
title:  "OWASP BWA WebGoat Challenge"
subtitle: "Introduction"
date:   2017-01-23 22:33:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
This is the first entry in a series of posts that will be focused on my working through the Open Web Application Security Project's WebGoat (v5.4). I do realize that there is a newer version (stable 7, dev 8?), but 5.4 came in the latest version of the OWASP Broken Web Applications Project. I intend to work my way through all of the challenges in the OWASP BWA over the course of the next few months after work and in my spare time. I will try to make enough progress to post meaningful updates to this blog on a daily to weekly basis.

A little bit about my setup:

I am going to be doing most of this on my 2015 Macbook Pro that I am single-booting Ubuntu 16.04 Xenial on. I have a couple quality of life improvements like macfanctld that keeps my CPU from frying and some window scaling adjustments to keep my eyes from bleeding everytime I try to read something on an app that tries to scale to the native Retina DPI (*cough* most Java-based apps). I still only get ~2hrs on a full charge, but I think that using Linux as my main OS instead of VM'ing forces me to learn and experiment more.

Most of my scripting and text editing is done in [Sublime Text 3][sublime-text-3], which is an awesome light ide. Some of the heavier project Python dev (like my very early-stage torrent client) I do in JetBrains' fully-fledged Pycharm IDE due to quality-of-life improvements like code-completion, error-checking, etc.

I'm running the [OWASP BWA 1.2][owasp-bwa] in Virtualbox on a Host-only network adaptor to keep myself from getting owned while I attempt to figure out what the heck I'm doing dabbling in the netsec world.

So I guess what I'm trying to say is,

{% highlight python %}
#!/usr/bin/env python
def main()
  print ("48656c6c6f2c20776f726c64".decode("hex"))

main()
{% endhighlight %}

[owasp-bwa]:https://sourceforge.net/projects/owaspbwa/files/
[sublime-text-3]:https://www.sublimetext.com/3