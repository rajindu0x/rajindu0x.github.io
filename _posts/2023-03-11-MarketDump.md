---
title: MarketDump - HTB Forensics
date: 2024-03-11 12.18 +0530
categories: [Digital Forensics, HackTheBox]
tags: [Forensics, PCAP, Telnet, Weak credentials ]
---

## Challenge Description: 
We have got informed that a hacker managed to get into our internal network after pivoting through the web platform that runs in public internet. He managed to bypass our small product stocks logging platform and then he got our costumer database file. We believe that only one of our costumers was targeted. Can you find out who the customer was? 

## Analysis: 
We are provided with a zip file. Upon unzipping we are given a pcap file. Let's open it with Wireshark.
![pcap analysis with wireshark](/assets/img/posts/marketdump/pcap_wireshark.png)

The pcap contains only 2868 packets. It should be easy to analyze :)
Let's take a look at protocol hierarchy by navigating to statistics > protocol hierarchy
![protocol hierarchy](/assets/img/posts/marketdump/protocol_hierarchy.png)

The most interesting protocol here is telnet. And we can see 1.6% of the packets are telnet. So let's first apply the protocol filter 'telnet'.

As there are only 46 packets, let's examine them manually. To view the telnet data we need to expand telnet protocol in packets details pain. If we inspect from the first packet to down, we can see that some Data have been parsed. This includes 'User:', 'admin', 'Pass:', 'Sorry, try again!' and finally 'Welcome, admin'. It's clear that the attacker tried to login with some usernames and passwords and at first the attempts failed. But eventually, the attacker used the correct combination of username and password and logged into telnet.

If we follow the tcp stream of the packet(stream 1053), that the telnet data says "Welcome, admin", we can see the attacker successfully logged in 
![telnet login](/assets/img/posts/marketdump/telnet_login.png)

By reading the next streams, we can see that in the stream 1055, the attacker establishes netcat listener on the machine and uses bash shell for code execution.
![initiating a netcat listener](/assets/img/posts/marketdump/nc_reverse_shell.png)

By navigating to next tcp stream(1056), we can see that the attacker executes several commands and the attacker is already root of the system.

```bash
ls -la
total 344
drwxr-xr-x 2 vigil vigil   4096 Jul  9 13:42 .
drwxr-xr-x 6 root  root    4096 Jul  9 13:38 ..
-rwxr-xr-x 1 vigil vigil 339920 Jul  9 13:24 costumers.sql
-rwxr-xr-x 1 vigil vigil    593 Jul  9 13:14 login.sh
pw
pwd
/var/www/html/MarketDump
ls -la
total 344
drwxr-xr-x 2 vigil vigil   4096 Jul  9 13:42 .
drwxr-xr-x 6 root  root    4096 Jul  9 13:38 ..
-rwxr-xr-x 1 vigil vigil 339920 Jul  9 13:24 costumers.sql
-rwxr-xr-x 1 vigil vigil    593 Jul  9 13:14 login.sh
whoami
root
wc -l costumers.sql
10302 costumers.sql
ls -la
total 344
drwxr-xr-x 2 vigil vigil   4096 Jul  9 13:55 .
drwxr-xr-x 6 root  root    4096 Jul  9 13:38 ..
-rwxr-xr-x 1 vigil vigil 333845 Jul  9 13:55 costumers.sql
-rw-r--r-- 1 root  root    1024 Jul  9 13:55 .costumers.sql.swp
-rwxr-xr-x 1 vigil vigil    593 Jul  9 13:14 login.sh
head -n2 costumers.sql
IssuingNetwork,CardNumber
American Express,377815700308782
cp costumers.sql /tmp/
cd /tmp
ls
config-err-lU04xV
costumers.sql
mozilla_vigil0
snap.1000_telegram-desktop_0UDXXk
ssh-8jVN4Kyx3X69
systemd-private-9ac4f21175984888b953531b43a88a47-apache2.service-lIsVqD
systemd-private-9ac4f21175984888b953531b43a88a47-bolt.service-Fd1LWs
systemd-private-9ac4f21175984888b953531b43a88a47-colord.service-rdNsnK
systemd-private-9ac4f21175984888b953531b43a88a47-fwupd.service-3d8iRg
systemd-private-9ac4f21175984888b953531b43a88a47-rtkit-daemon.service-pzu6lE
systemd-private-9ac4f21175984888b953531b43a88a47-systemd-resolved.service-ZtjIX4
systemd-private-9ac4f21175984888b953531b43a88a47-systemd-timesyncd.service-0BNKmh
Temp-bf8572b5-6aac-4c1d-aff6-063f56964ecb
python -m SimpleHTTPServer 9998
cat costumers.sql
IssuingNetwork,CardNumber
American Express,377815700308782
American Express,372184234300624
American Express,376615101453695
American Express,347640290681738

```
## Solution:

Attacker eventually finds out the file costumers.sql. Then the attacker executes a command to find out how many lines are there in the file, next reads the first two lines of the file. Upon discovering the file contains card details, the attacker copies the  file to /tmp folder and start a python http server to server the file out.

We now know the attacker, served the folder containing costumers.sql. The attacker should have done a GET request to /costumers.sql to exfiltrate  the file data. By using Wireshark's find tool, we can search for the string 'GET' in packet bytes.
![Get request to exfiltrate data](/assets/img/posts/marketdump/get_request.png)

By following the tcp stream we can see the attacker's ip address and also if we scroll down a little bit along the card numbers, we can find anomaly in card numbers. It seems to be encoded text.
![encoded flag](/assets/img/posts/marketdump/flag_encoded.png)

Upon decoding the text we are finally given the flag!
![decoded flag](/assets/img/posts/marketdump/flag_decoded.png)

## Lessons learned: 
The incident reveals vulnerabilities in network security, notably weak password usage in telnet access, leading to unauthorized entry and exfiltration of sensitive data. Key takeaways include the imperative of enforcing robust password policies, and monitoring network traffic rigorously to detect and prevent unauthorized access and data exfiltration. Additionally, it emphasizes the critical need for organizations to transition away from insecure protocols like Telnet and adopt more secure alternatives, such as SSH, to protect against unauthorized access and data breaches
