---
layout: post
title:  "OWASP BWA WebGoat Challenge: Injection Flaws"
subtitle: "String SQL Injection and Lab Exercises"
date:   2017-01-30 03:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# String SQL Injection
Instructions: 

- The form below allows a user to view their credit card numbers. Try to inject an SQL string that results in all the credit card numbers being displayed. Try the user name of 'Smith'. 

Alright. So we clearly cannot view all the users credit card numbers with an input of 'Smith'. Let's try to insert a string that will always evaluate to ```true```:

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_11/failed-injection.jpg">

Right, we forgot to escape our strings properly. Let's try that again:

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_11/injection-success.jpg">

Awesome!

# LAB: String SQL Injection
Instructions:

- Use String SQL Injection to bypass authentication. Use SQL injection to log in as the boss ('Neville') without using the correct password. Verify that Neville's profile can be viewed and that all functions are available (including Search, Create, and Delete).

Alright, let's try to login as neville and see what request is created:

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_11/neville-login.jpg">

```
employee_id=112&password=test&action=Login
```

Okay. Now let's think about what happens server side with our input. As we have both username and password here, it is likely that the evaluation looks something like this:

```
'select user from employees where employee_id=' + input.employee_id + 'and password=' + input.password '
```

While we do know Neville's employee_id, we don't have his password. If we appended our ```or true``` statement to his ```employee_id``` field, the password would still fail and we wouldn't be logged in. However, if we throw it on the password field, even an incorrect password should grant us access. Let's give it a shot:

```
employee_id=112&password=test' or '1' = '1&action=Login
```

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_11/string-injection-success.jpg">

Just as we thought!

# LAB: Numeric SQL Injection
Instructions:

- As regular employee 'Larry', use SQL injection into a parameter of the View function (from the List Staff page) to view the profile of the boss ('Neville').

Taking a look at the ```ViewProfile``` request, we see that it just takes in an ```employee_id```:

```
employee_id=101&action=ViewProfile
```

Now we need to get a little crafty with our SQLi. We can't pick Neville's profile directly, as we can only use the one statement we are crafting with our input. Appending another won't return the data we want as the shown output. Instead we can alter our statement to change the order of the returned data. The ```order by``` operator allows us to pick a column that our returned data will be sorted by. Here we have chosen salary. We added the additional ordering designation of ```desc``` to return the results in descending order. This assumes that Neville is paid the most numerically and lexicographically.

```
employee_id=101 or 1=1 order by salary desc&action=ViewProfile
```

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_11/numeric-injection-success.jpg">

But we were right! I'll be honest I did look at the hints for this challenge. SQL statements can get fairly complex relatively quickly. Luckily we can also make them very specific, and use the situational return data to tailor our output (or DB commands) to what we need.

### References

[1. SQL Injection Reference][owasp-sql]

[owasp-sql]:https://www.owasp.org/index.php/SQL_Injection