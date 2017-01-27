---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 3 (Final)"
date:   2017-01-24 18:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# AJAX Security Part 11: Insecure Client Storage Part 1
Instructions:

- STAGE 1: For this exercise, your mission is to discover a coupon code to receive an unintended discount.

First things first, lets look at the source and see if there is any javascript that will give us a clue as to what we're looking for. The entry field for a coupon has a function in it called ```isValidCoupon```. 

```
<input value="" onkeyup="isValidCoupon(field1.value)" name="field1" type="TEXT">
```

That seems like a good place to start. Looking at the JS files, we see one called ```clientSideValidation.js``` with the function ```isValidCoupon``` at the top of the file. Looks good! Let's inspect it a little closer:

```
var coupons = ["nvojubmq",
"emph",
"sfwmjt",
"faopsc",
"fopttfsq",
"pxuttfsq"];

function isValidCoupon(coupon) {
	coupon = coupon.toUpperCase();
	for(var i=0; i<coupons.length; i++) {
		decrypted = decrypt(coupons[i]);
		if(coupon == decrypted){
			ajaxFunction(coupon);
			return true
	}
	return false;	
}
```

Aha! a list of hardcoded coupons. Well, encrypted coupons.  Anyways, let's break down the logic of this function. First the input is coupon is converted to upper case. Then we loop through each of the strings in the ```coupons``` array. In each loop we call the ```decrypt``` function on the current coupon string. The decrypted coupon is then compared to the capitalized input. If they are the same, an AJAX request is fired off to the webserver (who responds with the discount amount) saying that says the coupon is valid and the function returns true. Otherwise, it returns false. 

The decryption method is a simple caesar cipher. However, we don't need to reverse engineer anything. If we hop into console we can just call ```decrypt``` on one of the hardcoded encoded coupons and use the output as input on the checkout page.

```
$decrypt("emph")
>"GOLD"
``` 

Let's try it. We enter "GOLD" (or any capitalization variation of those letters in the same order) and hit Purchase:

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_final/insecure-storage-1.jpg">

Nice!

# AJAX Security Part 12: Insecure Client Storage (Part 2)
Alright, now apparently we need to get our entire order for free. Let's look back at the ```clientSideValidation.js``` code and see what else we can find

```
function updateTotals(){

	f = document.form;
	
	f.TOT1.value = calcTot(f.PRC1.value , f.QTY1.value);
	f.TOT2.value = calcTot(f.PRC2.value , f.QTY2.value);
	f.TOT3.value = calcTot(f.PRC3.value , f.QTY3.value);
	f.TOT4.value = calcTot(f.PRC4.value , f.QTY4.value);	
	
	f.SUBTOT.value = formatCurrency(unFormat(f.TOT1.value) 
							+ unFormat(f.TOT2.value) 
							+ unFormat(f.TOT3.value) 
							+ unFormat(f.TOT4.value));
	
	f.GRANDTOT.value = f.SUBTOT.value;	
	
	isValidCoupon(f.field1.value);
}
```

Well, unfortunately for this webstore they calculate totals for individual items by multiplying the quantity (document.form.QTYn.value) by the price (document.form.PRCn.value). Both of which fields are under our (attackers) control. So we fill our cart up with goods, hop into ```InspectElement``` in our browser to update the price of each object to $0, and refresh the page.

<img src="{{ site.baseurl }}/images/2017-01-24-webgoat_part_3_final/insecure-storage-2.jpg">

The point is really getting driven home...

Don't. Do. Clientside. Validation.

### References

1 [Caesar Cipher][caesar]

[caesar]:https://learncryptography.com/classical-encryption/caesar-cipher