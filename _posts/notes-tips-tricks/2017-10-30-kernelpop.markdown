---
layout: post
title:  "kernelpop"
subtitle: "Personal Projects"
date:   2017-10-29 00:00:00 -0500
author: "coastal"
header-img: "images/site-resources/kernelpop-header.png"
---

I've found myself some more coding time and decided to tackle a problem that I've had on previous challenges that I found unnecessarily opaque and frustrating: kernel exploitation. Finding the running kernel on a given target OS and matching it to useable exploits and source code always seems to me to be more of an exercise in `google-fu` than anything else. So to attack the problem of time wasted googling, I am developing a tool [kernelpop](https://github.com/spencerdodd/kernelpop).

`kernelpop` is a framework for discovering kernel exploits to match a particular kernel version and operating system. While there are some tools out there to perform potential kernel version / exploit matching, none of them do more than matching kernel versions to exploits. While `kernelpop` also does this simple matching, it goes beyond and can check prerequisite conditions for specific exploits in order to more accurately determine a system's vulnerability state.

Take for example the sudo exploit from earlier this year `CVE-2017-1000367`. While it can be matched to a range of kernel versions from when the vulnerability was introduced to when it was patched, it has additional vulnerability requirements: 

	-sudo version < 1.8.21
	-selinux-enabled
	-sudo built with selinux support

Simple kernel version matching cannot determine if a system meets these requirements. But, `kernelpop` can with its `brute-enumerate` mode (`-b` flag):

<img src="{{ site.baseurl }}/images/notes-tips-tricks/kernelpop-sudo.png">

As you can see, `kernelpop` in the `brute-enumerate` mode runs through each of the prerequisite checks and determines if the underlying system satisfies the exploit's requirements. If it does, it returns information that the system is confirmed vulnerable to the given kernel exploit. If not, like in this case, it also alerts so that you don't waste time trying exploits that will fail (and might not tell you when you run the exploit source code).

In addition to the in-depth `brute-enumerate` mode, the default mode for `kernelpop` simply performs the kernel version number matching to the built-in exploits. There is also an `input` mode, where you can input the output of a `uname -a` command and `kernelpop` will try to find exploits for the inputted kernel.

<img src="{{ site.baseurl }}/images/notes-tips-tricks/kernelpop-input.png">

This makes `kernelpop` a great tool for an attacker who doesn't want to bring or run outside code on a target machine and still allows you to find potential exploits for the targeted kernel.

I have implemented part of the framework to also perform automated exploitation of the kernel. I haven't yet implemented this, and think it will take some more involved work with each of the exploits and source code. I think my next steps are going to be adding more exploits, and adding in other privilege escalation checks like checking for incorrect sudo permissions, unescaped file paths (windows), etc. to provide more complete support for priv esc outside of kernel exploits.

For now you can find the code on the my github (as linked above). If you have any questions, comments, or requests, please reach out!

Until next time...

-coastal

