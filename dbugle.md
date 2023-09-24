---
layout: default
title: Wekor
description: THM - Wekor
---

## Enumeration

Nmap Scan:
```
Nmap scan report for 10.10.8.229
Host is up (0.073s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
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
