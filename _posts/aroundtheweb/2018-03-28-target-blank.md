---
layout: post
title:  "target=' blank'"
subtitle: "around the web"
date:   2018-03-28 12:00:00 -0500
author: "coastal"
header-img: "images/site-resources/cuttlefish.jpg"
---

### intro

I came across an interesting vulnerability the other day. It has been talked about before, but was new to me and I found an exploitable example in the wild so I thought I would write a post about it. The vulnerability in question is the `target="_blank"` vulnerability.

### the vulnerability

I was messing about on a new social media site, [darto.com](https://darto.com), and went to the user profile page, where I have previously found some good stuff (see [CVE-2018-7663](https://spencerdodd.github.io/2018/03/06/cve-2018-7663/)). I noticed that users were allowed to set a `Website` field that was externally viewable by other members of the site. Investigating vulnerabilities around `href`s, I saw an interesting post about `target="_blank"`[here](https://dev.to/ben/the-targetblank-vulnerability-by-example). 

The gist of the vulnerability is that when a new tab is opened, it retains some access to the original page that initiated the tab opening via the `window.opener` JavaScript API. The API can be used to set the location of the origin page by setting the value of `window.opener.location` to the website of choice. It is readily apparent that this could be used to implement a potent phishing attack. A user clicks a link and their focus is switched to the tab that was opened. The page, now filling the browser, executes JavaScript that changes the content of the previous tab to any site it pleases. This switch happens on page-load and while the user is looking at the new tab, so any page refreshing is not readily apparent.

The vulnerability was very recently added to MITRE's Common Weakness Enumeration (CWE) database as [CWE-1022](https://cwe.mitre.org/data/definitions/1022.html).

### PoC || GTFO

The exploit itself is very simple. The PoC I used is as follows:

```
<script>
	if (window.opener) {
		window.opener.location = "https://spencerdodd.github.io/pocs/targetblankd/";
	}
</script>
```

That's it. My PoC exploit page (the one that will be placed in a vulnerable `href`) can be found at [`https://spencerdodd.github.com/pocs/targetblank`](https://spencerdodd.github.com/pocs/targetblank). My PoC simply redirects to another [PoC page](https://spencerdodd.github.com/pocs/targetblankd) highlighting the location change. Here is the exploit in action in two different vulnerable locations I found on the the website in question:

### user profile

<img src="{{ site.baseurl }}/images/aroundtheweb/targetblank/poc-user.gif">

### site-wide post

<img src="{{ site.baseurl }}/images/aroundtheweb/targetblank/poc-post.gif">


### fixes

This vulnerability stems from `hrefs` not having the `rel="noopener"` attribute set. Accompany all `href`s that will open in a new tab (i.e. `target="_blank"`), with either

* `rel="noopener"`

or

* `rel="noopener noreferrer"`

The latter of which offers protection for Chrome, Safari, Opera, and Firefox users, the former of which only offers protection for Chrome, Opera, and Safari users. This is what a vulnerable `href` looks like:

```
<a class="top-user_website" href="https://spencerdodd.github.io/pocs/targetblank/" target="_blank">https://spencerdodd.github.io/pocs/targetblank/</a>
```

and this is what a non-exploitable `href` looks like:

```
<a class="top-user_website" href="https://spencerdodd.github.io/pocs/targetblank/" target="_blank" rel="noopener noreferrer">https://spencerdodd.github.io/pocs/targetblank/</a>
```

### xss

As a quick aside, I tried a simple XSS payload:

```
window.opener.document.write("\x3c\x73\x63\x72\x69\x70\x74\x3e\x61\x6c\x65\x72\x74\x28\x31\x29\x3b\x3c\x2f\x73\x63\x72\x69\x70\x74\x3e")
```

aka.

```
window.opener.document.write("<script>alert(1);</script>")
```

However, this was denied as modifying the `document` object is considered a violation of the cross origin policy.

<img src="{{ site.baseurl }}/images/aroundtheweb/targetblank/xss-denial.png">

### timeline

* `2018-03-27`: discovery of vulnerability

* `2018-03-27`: disclosure to [darto.com](https://darto.com) admins

* `2018-03-27`: response from admins recognizing the vulnerability and intent to fix that night

* `2018-03-28`: vulnerability verified patched via `rel="noopener noreferrer"`

* `2018-03-28`: permission given to publicly disclose the vulnerability

### addendum

I thought this was a cool vulnerability, and now that I know about it, I've been seeing it all over the place. It is a potentially potent phishing vector, and probably something more people should know about. Until next time!

-coastal

### references

1. [dev.to writeup on targetblank vulnerablities](https://dev.to/ben/the-targetblank-vulnerability-by-example)

2. [darto.com](https://darto.com)

3. [why not to use "target=\_blank"](https://css-tricks.com/use-target_blank/)

4. [CWE definition](https://cwe.mitre.org/data/definitions/1022.html)