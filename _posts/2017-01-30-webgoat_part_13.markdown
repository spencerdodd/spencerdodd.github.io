---
layout: post
title:  "OWASP BWA WebGoat Challenge: Injection Flaws"
subtitle: "Database Backdoors"
date:   2017-01-30 05:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Database Backdoors Part 1: Execute Multiple Statements
Instructions:

- Use String SQL Injection to execute more than one SQL Statement. The first stage of this lesson is to teach you how to use a vulnerable field to create two SQL statements. The first is the system's while the second is totally yours. Your account ID is 101. This page allows you to see your password, ssn and salary. Try to inject another update to update salary to something higher

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_13/test-input.jpg">

While jsmith's salary goes down ours goes up. This first injection is going to be formatted just like the first part of the last challenge.

```
101;update employee set salary=100000 where userid=101--
```

We simply insert that into the ```User ID``` form, and on submission:

<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_13/sqli-success.jpg">

We have moved into a higher tax bracket.

# Database Backdoors Part 2: Inject a Backdoor
Instructions: 

- Use String SQL Injection to inject a backdoor. The second stage of this lesson is to teach you how to use a vulneable fields to inject the DB work or the backdoor. Now try to use the same technique to inject a trigger that would act as SQL backdoor, the syntax of a trigger is:
CREATE TRIGGER myBackDoor BEFORE INSERT ON employee FOR EACH ROW BEGIN UPDATE employee SET email='john@hackme.com'WHERE userid = NEW.userid
Note that nothing will actually be executed because the current underlying DB doesn't support triggers.

SQL Triggers are functions that sit in the database structure and are called when certain events happen in the database. The example trigger acts when a new employee is added to the database. When 'triggered' it updates the email of the new employee to the hacker's email, giving them control of the account. Here we will craft a trigger with the same structure as the example and inject it into the database

```
101;create trigger sneakySnek before insert on employee for each row begin update employee set email='snek@sneaky.exe' where userid=new.userid
```
<img src="{{ site.baseurl }}/images/2017-01-30-webgoat_part_13/trigger-inserted.jpg">

    --..,_                     _,.--.
       `'.'.                .'`__ o  `;__.      sneaky
          '.'.            .'.'`  '---'`  `
            '.`'--....--'`.'
              `'--....--'`


### References

[1. SQL Triggers][sql-triggers]

[sql-triggers]:https://www.tutorialspoint.com/plsql/plsql_triggers.htm