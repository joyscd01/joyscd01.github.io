+++
date = '2026-06-24T19:25:01+02:00'
draft = false
title = 'Netmon Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`15/02/2025`**

![Netmon.jpeg](/images/imgs_netmon/Netmon.jpeg)

---
# Introduction

**_Netmon_** is a **Windows-based Easy** host that focuses on exploiting an exposed network monitoring application called **PRTG Network Monitor**.

The exploitation path is very straightforward and relies heavily on classic enumeration techniques. It starts with an **anonymous FTP login**, which allows us to explore the file system, grab the **User flag**, and discover old configuration backup files.

By extracting credentials from these backups and applying a bit of logical **password guessing**, we can log into the **PRTG web interface**. From there, gaining a shell is just a matter of executing a known **Authenticated RCE** vulnerability, which grants us a direct **NT AUTHORITY\SYSTEM** shell.

---
# Techniques Used
- **Anonymous FTP Access**

- **Information Disclosure → Backup Files**

- **Authenticated RCE → Metasploit**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- netmon -T4
```

![nmap_p.png](/images/imgs_netmon/nmap_p.png)

I found some **open ports**, but 3 of them immediately captured my attention:
- **21**/tcp - FTP

- **80**/tcp - HTTP

- **445**/tcp - microsoft-ds (SMB)

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV netmon
```

![nmap.png](/images/imgs_netmon/nmap.png)

**Useful Information gathered**:

1. The **FTP** service allows **anonymous login**.

2. The Web Server is hosting **"PRTG Network Monitor"**.

3. The **SMB service** seems to be running an old version.


## FTP - Anonymous Login & User Flag

Since **FTP** allowed **anonymous access**, that was my very first target. I logged in without providing a password:

```bash
ftp netmon                                                                       
Connected to netmon.
220 Microsoft FTP Service
Name (netmon:fallingstar): anonymous 
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp>
```

With access to the file system, I started enumerating the directories. Navigating to the **`Users\Public`** folder led me straight to the **user flag**:

![ftp_user.png](/images/imgs_netmon/ftp_user.png)

After securing the **user flag**, I shifted my focus to the web application. I did some Google searches about **PRTG Network Monitor** to figure out how the application is structured and where its configuration files are typically stored by default.

With this knowledge, I went back to my active **FTP** session and navigated to the configuration path:
**`C:\ProgramData\Paessler\PRTG Network Monitor`**

![config.png](/images/imgs_netmon/config.png)

Here, I hit the jackpot. I found a backup file named **`PRTG Configuration.old.bak`**.
I downloaded it to my local machine using the **`get`** command and read through it.

Searching for keywords like **"password"**, I found a set of credentials:

![admin_creds.png](/images/imgs_netmon/admin_creds.png)

Recovered credentials: **prtgadmin:PrTg@dmin2018**

---
## HTTP - Web Enumeration

I went to **`http://netmon`** to log into the **PRTG** web application.

At first, the login failed. However, looking at the password (**PrTg@dmin2018**), it seemed highly likely that the **administrator** simply updated the password by incrementing the year.

I changed the year from **2018** to **2019**, trying:

**prtgadmin:PrTg@dmin2019**

**Note**: _The choice of **2019** wasn't a random guess. Since this machine was originally released on **HackTheBox** in **2019**, it was the most logical year to try for the password rotation._

It worked perfectly! Once inside, I checked the dashboard and noticed the specific version of the app running.

![webapp_logged.png](/images/imgs_netmon/webapp_logged.png)

Knowing the exact version and having **administrator** access, I opened **Metasploit** to search for available exploit modules.

---
# Initial Access & Privilege Escalation | PRTG Authenticated RCE

Inside **msfconsole**, I searched for **PRTG** exploits and decided to use the **`exploit/windows/http/prtg_authenticated_rce`** module.

This specific exploit takes advantage of an **OS command injection** vulnerability in the **PRTG Network Monitor** web interface, allowing an authenticated user to **execute arbitrary commands**.

I set up the required options (**RHOSTS, LHOST, LPORT, and the recovered HTTP cookies/credentials**):

![msfconsole1.png](/images/imgs_netmon/msfconsole1.png)

**Note**: _Because the **PRTG** service on this machine is running with high privileges, exploiting this web vulnerability doesn't just give us a low-privileged shell, it drops us straight into an administrative context._

I ran the exploit and instantly caught a reverse shell as:

**_NT AUTHORITY\SYSTEM_**

With full control over the machine, I navigated to **`C:\Users\Administrator\Desktop`** and grabbed the **root flag**:

![root.png](/images/imgs_netmon/root.png)

---
# Final Thoughts

**_Netmon_** is a fast and straightforward box, making it an excellent practice ground for certifications like **eJPT**.

While the initial phase relies on a classic **CTF** trope, leaving the entire **`C:\`** drive exposed via **anonymous FTP**, the discovery of old **`.bak`** files is a widely recognized misconfiguration in the cybersecurity field. It serves as a great educational reminder to always check for configuration leftovers and backup files during enumeration.

The password tweaking part (changing **2018** to **2019**) is also a brilliant touch. It perfectly simulates human laziness regarding password rotation policies. Finally, using **Metasploit** to turn administrative web access into a **`SYSTEM`** shell wraps the box up nicely.