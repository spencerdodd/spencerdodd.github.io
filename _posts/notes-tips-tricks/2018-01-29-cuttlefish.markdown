---
layout: post
title:  "cuttlefish host enumeration"
subtitle: "tools"
date:   2018-01-29 00:00:00 -0500
author: "coastal"
header-img: "images/site-resources/pfsense-code-exec-header.jpg"
---

### why do i need another scanning tool?

I am preparing to take Offensive Security's Penetration Testing with Kali (PWK) course followed by the OSCP, and saw that people in the past have written tools for performing automated scanning of network hosts (to apply to the labs/exam). Most all of the tools focus on performing initial `nmap` scans and suggesting followup service enumeration to perform based on the results. I decided to make a tool that performs these follow up scans concurrently, and outputs the results in a directory structure per-host.

### what is cuttlefish

<img src="{{ site.baseurl }}/images/notes-tips-tricks/cuttles.jpg">

Besides being an adorable cephalopod, cuttlefish is a program I wrote for two reasons:

* to have a tailored automated enumeration tool for PWK / OSCP

* to write something useful in `golang`

I've been wanting to try out `golang` for a while. Historically I haven't liked how other languages made concurrency such a headache and I figured an enumeration tool would be a great application to write in a simplified concurrent framework.

The tool itself functions by identifying running services on a target host with an initial `nmap` scan. Any identified services that have supported follow up scans within the app will be enumerated by these scans. The results are written to log files in the following way:

```
results
└── ip.address.of.target
    └── timepoint
        ├── cuttlemain.cuttlelog
        ├── followup-scan-1.cuttlelog
        ├── followup-scan-2.cuttlelog
        └── followup-scan-3.cuttlelog
```

All of the log files use the `.cuttlelog` extension because...why not...but they're all just plaintext files. The `cuttlemain` file mirrors the output you see on the terminal, which just relays scan progress info. Each of the `folloup-scan-*` files hold the output of the scans that run on the identified services. Here is a `gif` of the tool in action (from the [github](https://github.com/spencerdodd/cuttlefish) page). 

<img src="{{ site.baseurl }}/images/notes-tips-tricks/cuttlescan.gif">

### wrapup

It's a pretty straightforward tool, and fairly extensible (for me...the source might be a pain for someone else to expand on). I wrote it for personal use, but if people want to use it I can work on restructuring it for adding new scan modules and services. I should restructure it to be a bit more conforming to the `golang` recommended project format. But for now, it does everything I need.
Cheers!

-coastal












