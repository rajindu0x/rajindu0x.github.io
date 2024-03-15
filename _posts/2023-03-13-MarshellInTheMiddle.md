---
title: Marshal in the Middle - HTB Forensics
date: 2024-03-13 21.145 +0530
categories: [Digital Forensics, HackTheBox]
tags: [Forensics, WireShark, Zeek, IDS, DNS, TLS, Log Analysis]
---

## Challenge Description: 
The security team was alerted to suspicious network activity from a production web server. Can you determine if any data was stolen and what it was?

## Analysis:
The challenge provides us with a zip file. Upon unzipping we are given few files for analysis. 
![zeek generated log files](/assets/img/posts/marshell_in_the_middle/log_files.png)

By looking at the file tree we can see several log files in bro/ folder which generated from Zeek Intrusion Detection System (I've learnt usage of Zeek from TryHackMe SOC analyst-1 path. HTB academy SOC analyst path also covers usage of Zeek in 'Working with IPS/IDS' module.). Additionally, there is a pcap file, a file containing RSA private key and it's certificate and finally a secrets.log file containing TLS keys. 

Zeek is a comprehensive Network traffic analysis tool containing a lot of functionality for analyzing and monitoring network traffic with the help of it's extensive library of analyzers and it's own powerful scripting language. However as we are given the generated logs, we just need to analyze generated log files and use that information to find the intrusion using the provided pcap file.

Let's be clear with what each log file in bro/ folder contains. 


    -conn.log: TCP/UDP/ICMP connections observed on the network 
    -dns.log: Details about DNS activities observed on the network
    -files.log: Details about files transferred over the network
    -http.log: HTTP requests and responses observed on the network
    -packet_filter.log: Information related to packets that have been filtered by firewall rules
    -ssl.log: Details about the SSL/TLS handshakes, certificates exchanged, and other metadata associated with SSL/TLS sessions
    -weird.log: unusual or unexpected network activity generated based on predefined rules

First, let's analyze dns.log to identify the hosts on the network.
![dns.log](/assets/img/posts/marshell_in_the_middle/dns_log.png)
Here, we can identify 3 hosts: 10.10.20.1, 10.10.20.13 and 10.10.100.43. It's clear that 10.10.20.1 is the router and all of it's activity is done through port 53, which is default port for dns. We have 2 other hosts querying various site urls. Among them 10.10.20.13 happen to be a bit suspicious as it is querying for sql production database and pasetebin.com. The other host(10.10.100.43) requests normal sites such as google. reddit, youtube, linkedin etc. 

So, let's keep our focus on 10.10.20.13. By analyzing SSL log, we can confirm our suspicious host has accessed pastebin.com.
![ssl.log](/assets/img/posts/marshell_in_the_middle/tls_log.png)

If we analyze weird.log file, we can identify another host 10.10.99.42 on our network that our suspicious host tried to talk with.
![weird.log](/assets/img/posts/marshell_in_the_middle/weird_log.png)

The other 3 log files packet_filter.log, http.log and files.log doesn't seem interesting at this point as it doesn't contain information about our suspicious host.

Now, with this information let's analyze our pcap file to find out the intrusion. After opening the pcap file with WireShark, we can add a display filter for our suspicious host and the other host it talked with. 
![WireShark filtering](/assets/img/posts/marshell_in_the_middle/wireshark_filter.png)

## Solution:
Following TCP stream we can see the attacker got code execution as root on out suspicious host. The attacker used exfildb.sh to dump the production database 'mysql-m1.prod.htb' to a file named 'dbdump' and we can see that the attacker already knows the root password of our host. 
![Code execution](/assets/img/posts/marshell_in_the_middle/intrusion.png)

We earlier saw the suspicious host made requests to pastebin.com. Here, we can see exactly what he did with pastebin.com. By reading the stream, we can see the attacker first made a post request to pastebin.com with his api key and pasted /etc/passwd, /etc/shadow and finally the contents of dbdump file. Damn!

![Code execution](/assets/img/posts/marshell_in_the_middle/shredding_files.png)
Finally, the attacker shred the files he made in /tmp/.hfx folder and exits.
I tried to find attackers POST request interaction with pastebin.com using the display filter '(ip.addr==10.10.99.42 || ip.addr==10.10.20.13) && http.request.method==POST'. But the requests cannot be found using WireShark. In order to see the POST requests made, we need to use provided TLS secret file to decrypt the TLS traffic in WireShark. 

We can do this in WireShark by Edit > Preferences > Protocols > TLS and supplying secret.log file to (Pre)-Master-Secret Log file name. We have to tick both checkboxes containing Reassemble TLS...

Now, we can see 3 POST requests, the attacker made to pastebin.com
![Code execution](/assets/img/posts/marshell_in_the_middle/decrypted_TLS.png)

Following the final HTTP stream 'POST /api/api_post.php', We can see the contents of the exfiltrated database. Scrolling a bit down, we finally find our flag.


![Flag](/assets/img/posts/marshell_in_the_middle/flag.png)

## Lessons Learned:
This analysis demonstrates the importance of leveraging network analysis tools like WireShark and Zeek to identify suspicious activities, focus on relevant hosts, and decrypt encrypted traffic for deeper inspection. By analyzing log files and network traffic, the presence of a suspicious host engaging in unusual behavior was detected. Further investigation revealed post-exploitation activities, including unauthorized access, data exfiltration to Pastebin, and file shredding to cover tracks. This underscores the necessity of continuous improvement in cybersecurity skills and thorough post-incident analysis to understand and respond effectively to security incidents.








