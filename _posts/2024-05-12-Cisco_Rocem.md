---
title: My University's Switch DOS - Project
date: 2024-05-12 18:02:00 +0330
categories: [Project]
tags: [hardware,cve]
description: Cisco Catalyst 2960 Denial of System

---

# - **Introduction**
As you may know, we are surrounded by electronics and like it or not they make life a hell of a lot easier. Especially routers and modems which grant us internet access.  
Now, what if I tell you that every single one of them is flawed and has a weakness? That is what I love about them! To look around and find a way to exploit those weaknesses.
A very specific device that I came across and is actually the topic of our post today is the **Cisco Catalyst 2960** switch.  
In my university, department of basic sciences to be percise, there are clusters of this switch and many **ALTAI** outdoor access points scattered around the building and one day I got curious.  
So I ran an Nmap scan of the switch and analyzed the traffic via Wireshark and I found out that the IOS version is **12.2**. After some research and testing, I dicovered that it is vulnerable to **CVE-2017-3881**.

# - **How does the exploit work**
Improper validation of input data in the Cisco CMP (Cluster Management Protocol) leads to the vulnerability. This can allow the attacker to send specially crafted packets through telnet to the switch, leading to a denial of service.  
As it can be seen in [ExploitDB](https://www.exploit-db.com/exploits/41872), it sets a credless privilege 15 authentication to the switch which will cause a DOS, if not we execute commands. Unfortunately here it caused the system to reboot, but I plan to improve the exploit so that we can have the lovely RCE. For now, the DOS actually came pretty handy several times, for example I could skip to the front of queues behind office doors just because managers told the students the system went offline.

# - **POC**

![update](assets/img/rocem3.jpg)

>As I did not have my laptop with me at the time of testing the exploit and I was on termux, I used Metasploit as it was faster.
{: .prompt-info }

Thanks for reading!