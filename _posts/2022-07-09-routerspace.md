---
title: "Routerspace Hackthebox writeup"
author: "0xMesbaha"
cover: "/media/routerspace/20220707205351.png"
tags: ["Hackthebox", "apk", "adb", "sudo-exploit"]
date: 2022-07-09T00:51:12+02:00
draft: false
---

In this Box we are going to examine an android appliacation (apk) , and monitroing the requests by placing a proxy we will notice a request which we can manipulate to get a remote code exection. this box has a lot of iptables rules which restrict us from getting a reverse shell in the usual way. so we will login via ssh by placing our public key at paul's. from that we will gain root access by exploiting sudo itself.

<!--more-->

# Scanning

```bash
nmap -A -T4 $IP -oN nmap.intial
```

Useful output

```shell
PORT   STATE SERVICE VERSION
22/tcp open  ssh     (protocol 2.0)
80/tcp open  http
```

# Enumeration

visiting the web application at port 80 we can see :

![error](/media/routerspace/20220707205828.png)

if we focus on the images we can see the hostname `routerspace.htb`

![error](/media/routerspace/20220707210233.png)

However it seems it is the only host on that IP , so it was not really useful :D

the only interesting thing at this page is the download option , we can download it

it seems like an apk file , we need to open it so we can examine its functions.

we can do that by installing `anbox` which can help running APK on Linux. But note that we also need to set a proxy so if any requests are issued we should be aware of it.

-> install [anbox](https://www.how2shout.com/linux/how-to-install-anbox-on-ubuntu-20-04-lts-focal-fossa/) "works on ubuntu and not on kali"

```bash
adb install RouterSpace.apk

adb shell settings put global http_proxy <tun0-ip>:8081
```

we also need to set this setting in burpsuite

![error](/media/routerspace/20220707211353.png)

now if we open `anbox` we should see the application

![error](/media/routerspace/20220707211728.png)

clicking on the `check status ` multiple times we can see this request issued

![error](/media/routerspace/20220707211938.png)

![error](/media/routerspace/20220707212014.png)

if we forward the request we can see

![error](/media/routerspace/20220707212509.png)

after trying different types of inputs (strings, numbers) it looks like it will reflect whatever data we will give it

Trying some basic kind of remote code execution

![error](/media/routerspace/20220707213014.png)

and we have remote coded execution indeed

# Foothold

- now we want to get a reverse shell back , we must be cautious of whatever payload we are passing because it may conflict with JSON parser and other stuff.

- However trying to avoid all the special character still no connection call back, as we can't generate ssh keys for paul as this requires an interactive steps we can generate ssh pairs and place the public key inside his `~/.ssh/authorized_keys` file.

```bash
# @ our side
ssh-keygen  -f routerpsace
```

![error](/media/routerspace/20220707215209.png)

we can confirm it is placed with no issues by catting the file latter.

```bash
ssh paul@routerspace.htb -i routerpsace
```

and we are in

![error](/media/routerspace/20220707215309.png)

# Privilege Escalation

After some of my checklists `(sudo -l , suids , examining un unusual files, ..)` . you can figure out by your own or by any automated tool that the `sudo` version is vulnerable to local privilege escalation

```bash
sudo --version
Sudo version 1.8.31
Sudoers policy plugin version 1.8.31
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.31

```

and we can find exploits on github like this one [here](https://github.com/mohinparamasivam/Sudo-1.8.31-Root-Exploit)

it seems there are IPTABLE rules that doesn't permit outband communication so we can't use curl,wget . we can use `scp` instead

```bash
# @ our side
tar cvfz exploit.tar Sudo-1.8.31-Root-Exploit
scp -i routerpsace exploit.tar paul@routerspace.htb:/dev/shm/.my-file/exploit.tar
```

at the machine side :

```bash
tar xf exploit.tar
cd Sudo-1.8.31-Root-Exploit/
make
```

![error](/media/routerspace/20220707225546.png)

## pwned
