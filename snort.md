---
layout: default
title: Snort
description: Setting Up Snort IDS (NIDS in this case)
---

Today we'll set up Snort IDS on a Ubuntu host, on the same network as our Windows 10 victim that we used in the BitDefender bypass post.

Before I did that, I set up a bridged network using [this guide](https://www.redhat.com/sysadmin/setup-network-bridge-VM).

Now that we have our network setup, let's install snort.

First step:
```
sudo apt-get install snort -y
```
Make sure you set up the device and the correct CIDR. In my case, it was enp1s0, with 192.168.0/24.

After snort is installed:
```
cat /etc/snort/snort.conf
```
and edit the "HOME_NET" line, to point to your CIDR.

![01](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/01_Snort_Home_net_setup.png)

After this, you must set up promiscuous mode on your ubuntu machine.

![001](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/001_Snort_Version_And_Promisc.png)

When that was also done, for my testing purposes, I commented out the ICMP rules in the snort.conf file, so we can add our own.

![02](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/02_Comment_out_ICPM.png)

And now we can test our snort configuration, by running:
```
sudo snort -T -i <interface> -c /etc/snort/snort.conf
```
And the results are as follows:

![03](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/03_Snort_Config_Testing.png)

![04](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/04_Snort_Config_Testing_Rules_Loaded.png)

![05](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/05_Snort_Config_Testing_Success.png)

The rules were loaded, and our config file works.

Now let's get to the demo, where first I'll ping the windows 10 machine without any ICMP rules, and after that, I'll add an ICMP rule to see if I catch it.

![06](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/06_windows_and_snort_windowsip_noping_rule.png)

![07](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/07_ICMP_No_alert.png)

As you can see, we didn't get any alerts regarding the pings.

Now we'll add our rule, and see if we can catch it.

You can add your own rules to "/etc/snort/rules/local.rules"

![08](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/08_ICMP_Rule_Set.png)

And now if we save it, re-run snort, and ping the machine again:

![09](https://raw.githubusercontent.com/TudM99/tudm99.github.io/main/images/09_ICMP_Ding_Dong.png)

Bingo! We caught the ping.

Now, this is just a small example. The community rules can protect your infrastructure from CVE abuse, like Eternal-Blue, and all kinds of other nasty stuff.

Depending on how you set it up, you can configure Snort to drop the packet, and thus protect the victim PC. 

It's a very robust system, with a lot of customization potential.

