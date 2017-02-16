---
layout: post
title:  "OWASP BWA WebGoat Challenge: Injection Flaws"
subtitle: "Modify and Add Data with SQL Injection"
date:   2017-01-30 04:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Modify Data with SQL Injection
Instructions:

- The form below allows a user to view salaries associated with a userid (from the table named salaries). This form is vulnerable to String SQL Injection. In order to pass this lesson, use SQL Injection to modify the salary for userid jsmith.

It looks like our rival pentester is taking home some cash!


<img src="{{ site.baseurl }}/images/webgoat/2017-01-30-webgoat_part_12/sqli-initial.jpg">

Unfortunately for jsmith, we're not a huge fan. Let's see if we can knock that salary down a peg or two. Here we are going to use the ```update``` command. This command allows for the alteration of existing data in a SQL table. Let's come up with an injection that will alter this existing data.

```
jsmith';update salaries set salary=9001 where userid='jsmith'--
```

There are two subtle pieces to this injection that are essential to its success and the ability to combine more than one statement given the way that our input is handled. First is the ```';``` before our ```update``` call. This truncates the previous statement and allows us to add our tailored trailing statement. The other piece is the ```--``` at the very end. This escapes any remaining data that was a part of the server's intended statement. It allows us to call our second statement without throwing an error. The meat of our injection is an update that changes the salary of the user, whos ```userid``` is equal to 'jsmith', to 9001.

<img src="{{ site.baseurl }}/images/webgoat/2017-01-30-webgoat_part_12/sqli-data-modified.jpg">

And we've done it! Although his salary is still over 9000.

# Add Data with SQL Injection
Instructions:

- The form below allows a user to view salaries associated with a userid (from the table named salaries). This form is vulnerable to String SQL Injection. In order to pass this lesson, use SQL Injection to add a record to the table.

The last challenge had us update existing data, now we are going to create new data. Instead of using ```update```, we are going to leverage the ```insert``` command to create a new user in our table. We perform the same truncation as in the first injection, and follow it with an ```insert``` call that follows the format:

```insert into``` table ```(```row titles of replaced values```) values (```the values of those rows```)```

Our injection looks like this:

```
jsmith';insert into salaries (userid, salary) values ('anotherone', '100000')--
```

After we have injected our insertion, we can confirm that the command worked by using another injection:

```
anotherone';select * from salaries--
```

<img src="{{ site.baseurl }}/images/webgoat/2017-01-30-webgoat_part_12/sqli-data-added.jpg">

And we've got it!