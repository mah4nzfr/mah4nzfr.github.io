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

After looking up Grafana, there was this [exploit](https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit) which looked promising, but I needed credentials.