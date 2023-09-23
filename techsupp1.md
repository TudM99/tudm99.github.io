---
layout: default
title: Tech_Support:1
description: THM - Tech_Supp0rt:1
---

## Enumeration

Fired up nmap and gobuster.

```
Nmap scan report for <IP>
Host is up (0.067s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: TECHSUPPORT; OS: Linux; CPE: cpe:/o:linux:linux_kernel

```
```
/wordpress       (Status: 301) [Size: 316]
/test            (Status: 301) [Size: 311]
/server-status   (Status: 403) [Size: 277]
```
### SMB Enumeration:
```
 Disk                            Permissions
----                             -----------
print$                            NO ACCESS
websvr                            READ ONLY
IPC$                              NO ACCESS
```
Before checking out the web pages, I'll access the “websvr” share and see if there's anything juicy inside.

```
smbclient -N //<IP>/websvr

smb: \> ls
  .                                   D        0  Sat May 29 10:17:38 2021
  ..                                  D        0  Sat May 29 10:03:47 2021
  enter.txt                           N      273  Sat May 29 10:17:38 2021

                8460484 blocks of size 1024. 5693200 blocks available
smb: \> get enter.txt
getting file \enter.txt of size 273 as enter.txt (1.0 KiloBytes/sec) (average 1.0 KiloBytes/sec)
smb: \> exit

cat enter.txt
GOALS
=====
1)Make fake popup and host it online on Digital Ocean server
2)Fix subrion site, /subrion doesn't work, edit from panel
3)Edit wordpress website

IMP
===
Subrion creds
|->admin:[REDACTED] [cooked with magical formula]
Wordpress creds
|->
```

Now this is interesting. We found Subrion credentials. I'll throw them into Cyberchef and see what pops up.

![CyberChef](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/1.png)

[back](./)
