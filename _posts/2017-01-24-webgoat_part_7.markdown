---
layout: post
title:  "OWASP BWA WebGoat Challenge: Concurrency"
subtitle: "Thread Safety"
date:   2017-01-24 19:02:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Concurrency: Thread Safety Problems
Instructions:

- The user should be able to exploit the concurrency error in this web application and view login information for another user that is attempting the same function at the same time. This will require the use of two browsers. Valid user names are 'jeff' and 'dave'.

Let's take a peek at how the request to the server is formed:

```
username=jeff&SUBMIT=Submit
```

Well, nothing special there going on. So, let's queue up requests for both Jeff and Dave in separate browsers and set Burp up to intercept. Once Burp has caught both requests, we release them both as quickly to simultaneously as possible to maximize any problematic concurrency window. 

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_7/thread-success.jpg">

Burp shows me that the request for Jeff went in first, followed by the request for Dave. Both windows received Dave's information.

Here is some psuedocode of the server side code for this challenge:

```
current_user = None

def user_login(new_user):
	if current_user is None:
		current_user = new_user
		thread.sleep(1.5)
		return current_user.info
	else:
		current_user = new_user
		return current_user.info

```

Server side, what happens is this. The first user logs in and sets the global variable of ```current_user``` to themselves and enters a thread sleep for 1500 ms (1.5 seconds). After the sleep, the user returns the info of the ```current_user```, which it assumes to be them. However, while the first user was sleeping, the second user logged in. The ```user_login``` function then assigned the value of ```current_user``` to the value of the second user, and returned that data to the second user. When the first user wakes up from the thread sleep, it returns the info for the ```current_user``` which it assumes to be itself. However, the second user changed the global variable value to itself while the first user was sleeping, causing them both to return the values for the second user that logged in.

This is why you need to take careful consideration into designing threaded programs with shared resources, and ensure that this sort of concurrency issue doesn't arise.

# Concurrency: Shopping Cart Concurrency Flaw
Instructions:

-  For this exercise, your mission is to exploit the concurrency issue which will allow you to purchase merchandise for a lower price.

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_7/cart-init.jpg">

Alright, lets add one of each item to our cart and check all of the traffic in Burp. First thing to do is press ```Update Cart```:

```
QTY1=1&QTY2=1&QTY3=1&QTY4=1&SUBMIT=Update+Cart
```

The server response is just updated HTML for the browser. Now let's click ```Purchase```:

```
QTY1=1&QTY2=1&QTY3=1&QTY4=1&SUBMIT=Purchase
```

This takes us to a page where we submit our credit card information, and then press ```Confirm``` to presumably process our purchase. Let's see what it sends to the server:

```
CC=5321+1337+8888+2007&PAC=111&SUBMIT=Confirm
```

Notice that there is no order information in the confirmation response. That seems like it could be exploitable! Let's try to purchase an order for one item, get to the credit card confirmation page, but then update the order to contain a bunch more than our initial order. If there are no further checks server side for order confirmation, we may be able to get all of those items for the price of the initial order.

So we fill out our cart with one hard drive, update the cart, and move to checkout. 

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_7/cart-full.jpg">

Once we're here, we pull up the old request from ```Update Cart``` for 1 of each item in Burp and fire that off to the webserver. 

```
QTY1=1&QTY2=1&QTY3=1&QTY4=1&SUBMIT=Update+Cart
```

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_7/cart-fuller.jpg">

Then, back in our confirmation page, we press ```Confirm``` and...

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_7/cart-backtraced.jpg">

Too bad I was hiding behind 7 proxies.




*Side Note*:

I am finally caught up with all the writeups I had from the last couple days. I will be posting new updates as I finish them.