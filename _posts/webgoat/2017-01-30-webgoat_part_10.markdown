---
layout: post
title:  "OWASP BWA WebGoat Challenge: Injection Flaws"
subtitle: "Numeric Injection"
date:   2017-01-30 00:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Numeric Injection
Instructions:

- The form below allows a user to view weather data. Try to inject an SQL string that results in all the weather data being displayed. 

SQL injection is a common exploitation method that involves sending a malicious string to unfiltered (or poorly filtered) input parameters for database queries. On this page, we can see that we can select a weather station from a drop-down list. When we send our station request, we see that the request is formatted:

```
station=101&SUBMIT=Go%21
```

The request returns the following page that contains the response data from our query (in this case, the weather at the Columbia station (station number 101).

<img src="{{ site.baseurl }}/images/webgoat/2017-01-30-webgoat_part_10/num-home.jpg">

Alright so we know that the SQL query is formatted like so, thanks to some output on the response page:

```
SELECT * FROM weather_data WHERE station = []
```

This appears to be vulnerable to a simple SQL injection attack where we attach an 'or' operator followed by a statement that is always true. Now when the query goes through the data table, it compares every entry to our query where the ```station``` matches our input station, OR true. So whether or not the actual comparison matches, the query will always evaluate to true and we receive all the results of the table.

```
SELECT * FROM weather_data WHERE station = 101 OR TRUE
```

<img src="{{ site.baseurl }}/images/webgoat/2017-01-30-webgoat_part_10/num-inject-success.jpg">

Alright! 