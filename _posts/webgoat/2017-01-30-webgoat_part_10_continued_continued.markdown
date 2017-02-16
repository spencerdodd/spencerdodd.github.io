---
layout: post
title:  "OWASP BWA WebGoat Challenge: Injection Flaws"
subtitle: "XPATH Injection"
date:   2017-01-30 02:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# XPATH Injection
Instructions: 

- The form below allows employees to see all their personal data including their salaries. Your account is Mike/test123. Your goal is to try to see other employees data as well.

XML is becoming a somewhat increasingly popular database format. Data selection and querying of XML-based databases is performed with the XPATH query language. There are a few attacker advantages with XPATH-based injection vs. SQL-injection. For one, the implementation is identical universally. This means that we can create XPATH-based injections that will work on any DB using XPATH as a query language. With SQL, our injections will only work on the specific dialect of SQL that is used. Additionally, there are no Access Control Lists (ACLs) in XPATH. ACLs specify which users are allowed to perform which actions on what data in SQL. By not implementing this control in XPATH, our injections can access any data in the XML that we please giving our attacks much easier access to sensitive data.

Let's take a look at our login page:

<img src="{{ site.baseurl }}/images/webgoat/2017-01-30-webgoat_part_10_continued_continued/xpath-injection-start.jpg">

Now XPATH queries are formatted very similarly to SQL queries. Let's look at an example from [OWASPs page on XPATH Injection][owasp-xpath]:

Our data table is formatted like so:

```
<?xml version="1.0" encoding="ISO-8859-1"?> 
<users> 
<user> 
<username>gandalf</username> 
<password>!c3</password> 
<account>admin</account> 
</user> 
<user> 
<username>Stefan0</username> 
<password>w1s3c</password> 
<account>guest</account> 
</user> 
<user> 
<username>tony</username> 
<password>Un6R34kb!e</password> 
<account>guest</account> 
</user> 
</users> 
```

So, a query that returned the ```gandalf``` user from the table ```users``` would look like this:

```
string(//user[username/text()='gandalf' and password/text()='!c3']/account/text()) 
```

Now, we can format an injection attack just like in sql that will use string escapes to truncate our responses and add in our always true query appends:

```
user: ' or '1' = '1
pass: ' or '1' = '1
```

Which would format to:

```
string(//user[username/text()='gandalf' or '1' = '1' and password/text()='!c3' or '1' = '1']/account/text()) 
```
Returning all of the user values in the table ```users```.

When we try the same simple input for our XPATH injection:

<img src="{{ site.baseurl }}/images/webgoat/2017-01-30-webgoat_part_10_continued_continued/xpath-injection-success.jpg">

Nice! We were able to pull all of the user data from the XML-based database.

### Resources

[1. OWASP XPATH Injection][owasp-xpath]

[2. Access Control Lists][acl-sql]

[owasp-xpath]:https://www.owasp.org/index.php/Testing_for_XPath_Injection_(OTG-INPVAL-010)
[acl-sql]:http://stackoverflow.com/questions/3430181/how-can-i-implement-an-access-control-list-in-my-web-mvc-application