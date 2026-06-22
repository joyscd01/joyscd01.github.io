+++
date = '2026-05-19T21:10:41+02:00'
draft = false
title = 'Beep Writeup EN'
+++
**Name:** **`joy.scd01`**

**Date:** **`03/02/2025`**

![Beep.png](/images/imgs_beep/Beep.png)

---
# Introduction

**_Beep_** is an easy **Linux** machine designed for practicing different approaches to basic **penetration testing**.
I personally exploited this box through a **Local File Inclusion** vulnerability, but there are multiple valid attack paths, such as :

- **LFI through graph.php**
- **CVE-2012-4867 | vTiger CRM 5.1.0 LFI**
- **Webmin 1.570 RCE (Metasploit)**
- **CVE-2012-4869**

---
# Enumeration

## Nmap

Initial scan on all ports:

```bash
nmap -p- beep
```

![nmap1.png](/images/imgs_beep/nmap1.png)

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -T4 beep
```

![nmap2.png](/images/imgs_beep/nmap2.png)

**Primary points of interest**:

- **80**/tcp http

- **443**/tcp ssl/http

---
## HTTP - Web Enumeration | http://beep:80

The web page hosts an **Elastix** login interface.
For some reason, I had to change **`security.tls.version`** in **Firefox** to make the page visible.

Since the application didn’t disclose useful version information, I simply searched for **elastix** on **Searchsploit**:

![searc.png](/images/imgs_beep/searc.png)

Because **XSS** required an authenticated user, I focused instead on the **`graph.php` LFI** vulnerability.

_**`graph.php`** is vulnerable to **Local File Inclusion** because it fails to properly sanitize user input.
Using crafted path traversal payloads, attackers can retrieve **local files** and even **execute scripts** on the server._

---
# Initial Access |  LFI through graph.php

I tested the suggested payload:

```text
/vtigercrm/graph.php?current_language=../../../../../../../../etc/amportal.conf%00&module=Account&action
```

![lfi.png](/images/imgs_beep/lfi.png)

**(Viewing the page source make the output much more readable)** :

![creds.png](/images/imgs_beep/creds.png)

The retrieved file contained **credentials**.

I also used the LFI to read the **`/etc/passwd`** file, extracted all users into a wordlist, and tried brute-forcing with **Hydra**.
However, some protection seemed to throttle brute-force attempts.
Since the number of users and passwords was limited, I simply tested combinations manually.

---
## SSH Issue

When testing credentials via SSH, I encountered a problem with unsupported **key exchange algorithms**:

![algoerror.png](/images/imgs_beep/algoerror.png)

**Note**:
_This happens because **Beep** is an old machine, and I'm running it in 2025 on a modern **Kali Linux Purple** installation.
The SSH server only supports outdated algorithms that are no longer enabled by default on modern clients.
**To fix this**, I manually specified older algorithms compatible with the server._

---

The password **`jEhdIekWmdjE`** turned out to be valid for the **root** user.
After connecting, I obtained full control over the system:

```bash
ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa root@beep
```

![root.png](/images/imgs_beep/root.png)

At this point I was able to collect **both flags**.

---
# Final Thoughts

_Beep_ is a great box for anyone taking their first steps into **penetration testing**.
The attack vectors are simple but varied, which gives you the chance to practice multiple techniques.
It also helps you get into the real pentesting mindset: there’s rarely just one way in. Instead, you learn to explore different paths, test the security of each exposed service, and choose the most effective method to compromise the system.