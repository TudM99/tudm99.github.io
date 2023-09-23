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
/wordpress           	 	(Status: 301) [Size: 316]
/test                 				(Status: 301) [Size: 311]
/server-status        	(Status: 403) [Size: 277]
```

[back](./)
