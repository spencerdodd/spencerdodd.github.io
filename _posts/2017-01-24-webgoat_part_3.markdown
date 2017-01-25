---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 3"
date:   2017-01-24 16:00:00 -0500
categories: webgoat
---
# AJAX Security Part 1: DOM-based Cross Site Scripting
Alright, the last set of challenges seemed a little easy. Here's one that's going to be a little more involved and requires a bit of reading. For starters, we need to find out what AJAX is.

So AJAX stands for Asynchronous Javascript and XML. The protocol itself basically consists of using XMLHttpRequest objects to communicate with serverside scripts. It can utilize a number of transmittable data formats including JSON, XML, HTML, or text. AJAX is particularly attractive to web-development as it is asychronous and can update without page refreshes.

Next let's take a look at DOM. DOM stands for Document Object Model. It is a cross-platform, language-independent application programming interface for interacting with HTML, XHTML, or XML documents as a tree structure where each node is an object representing a part of the document. These objects can be manipulated programmatically, and any visible changes occurring as a result may then be reflected in the display of the document. In this challenge, we will be manipulating DOM fields through injections in forms that are fed to the webserver via AJAX requests and displayed back onto the webpage.

So our first goal is to deface the landing page for this challenge with the image found at ```http://192.168.56.101/WebGoat/images/logos/owasp.jpg``` (In v5.4 the link is broken; it is given as 'webgoat' instead of 'WebGoat'). To achieve this we can take a look at what the page does when we input some test string:

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3/dom-input.jpg">

After some further reading on XSS (cross site scripting), the attack vector becomes clear. Notice how our input is displayed back to us on the page? This means that, without proper sanitization, we can make the page display whatever we want it to. In fact, we could even insert JavaScript and make the page *do* whatever we wanted it to. 

Let's take a look at the source code to fully understand what's going on:

```
<form accept-charset="UNKNOWN" method="POST" name="form" action="attack?Screen=49&amp;menu=400" enctype="">
	<script src="javascript/DOMXSS.js" language="JavaScript"></script>
	<script src="javascript/escape.js" language="JavaScript"></script>
	<h1 id="greeting">Hello, test!</h1>
	Enter your name:
	<input value="" onkeyup="displayGreeting(person.value)" name="person" type="TEXT">
	<br>
	<br>
	<input name="SUBMIT" value="Submit Solution" type="SUBMIT">
 </form>
```
So the input that we place into the ```person``` input field is fed to the method ```displayGreeting``` onkeyup (aka, when we've finished typing). Let's look at that method:

{% highlight javascript %}
function displayGreeting(name) {
	if (name != ''){
		document.getElementById("greeting").innerHTML="Hello, " + name+ "!";
	}
}
{% endhighlight %}

Alright, so our form information is being fed to this javascript function that replaces the value of an element named ```greeting``` with the input. If we look back at the first code block, we see that the element greeting is the fourth line in the form. Good news for us, and the great news is that displayGreeting doesn't perform any input sanitization!

So let's see if we can deface this page by just inserting a valid html image tag in our form with the path we defined before. Our input looks like this:

```
<img src=http://192.168.56.101/WebGoat/images/logos/owasp.jpg> 
```

and ...

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3/dom-xss.jpg">

XSS'd!

# AJAX Security Part 2: DOM-based Cross Site Scripting (JavaScript alert)
Cool cool. Now that we've popped an image XSS it's time to try the most classic of XSS injections, the javascript alert pop. Javascript is natively interpreted by most all modern web browsers. While this is convenient for using and navigating responsive and pretty webpages, it also adds a tool to our web exploitation toolkit. In HTML, javascript is bounded by ```<script></script>``` tags. So let's change our image tag to a script tag, change our payload to an alert (which pops a dialog box to the user), and throw it into our input.

```
<script> javascript:alert("XSS")</script>
```

and ...

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3/dom-javascript-fail.jpg">

Huh...okay. Well, lucky for us there are tons of resources on getting XSS alerts through various forms. Referencing the [XSS filter evasion sheet][xss-fuzzing], we see a number of options. The one that I ended up going with was an image tag manipulation that pops using the ```onerror``` specification. You form this attack like so:

```
<img src=(resource that doesn't exist) onerror="alert('XSS')"></img>
```

and...

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3/dom-javascript-success.jpg">

Classic! There are a huge variety of ways to form an attack like this on a susceptible target. Take a look at the resource I linked earlier for an idea of how creative you can get with XSS.

# AJAX Security Part 3: DOM-based Cross Site Scripting (IFrame)
For this challenge we need to create a js based alert by embedding it in an IFrame tag. An IFrame is short for inline frame. IFrames are used to embed another document within the HTML the IFrame is present in.

Let's try to just wrap some js code in an IFrame and see what we can do:

```
<iframe src="javascript:alert('XSS');"></iframe>
```

and, when we place our crafted IFrame in the form:

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3/dom-iframe.jpg">

Frame'd

# AJAX Security Part 4: DOM-based Cross Site Scripting (Fake Login Prompt)
Now we're getting to more nefarious implementations of XSS. Our goal with this challenge is to show a fake login prompt for credential harvesting. We are given the HTML code that we need to show to fake this prompt. Now we need to package it into an XSS vector. Nice for us that IFrame has a builtin attribute for packaging source HTML code into a frame, called ```srcdoc```.

Let's wrap our HTML in an IFrame. Be careful we properly escape all of our internal strings, while still wrapping the entirety of the code as a string itself.

```
<iframe srcdoc="Please enter your password:<BR><input type = 'password' name='pass'/><button onClick='javascript:alert(`I have your password: ` + pass.value);'>Submit</button><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR>"</iframe>
```

Now we feed our payload to the vulnerable page, and ...

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3/dom-iframe-login.jpg">

Success! Although you'd have to be less aware than a rock with an ethernet jack to fall for that prompt, carefully tailored payloads can look pretty convincing:

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3/convincing-credential-harvester-1.jpg">
<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3/convincing-credential-harvester-2.jpg">

[Source][pentest-lab]

### Resources

1 [Asynchronous JavaScript and XML (AJAX)][ajax]

2 [Document Object Model (DOM)][dom]

3 [Cross Site Scripting (XSS)][xss]

4 [OWASP XSS Filter Evasion)][xss-fuzzing]

5 [IFrames][iframes]

[ajax]:https://developer.mozilla.org/en-US/docs/AJAX/Getting_Started
[dom]:https://www.w3.org/DOM/
[xss]:https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)
[xss-fuzzing]:https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet
[pentest-lab]:https://pentestlab.blog/2012/02/24/credential-harvester-attack-method/
[iframes]:http://www.w3schools.com/tags/tag_iframe.asp