+++
date = '2026-05-20T17:53:54+02:00'
draft = false
title = 'Cap Writeup EN'
+++
**Name:** **`joy.scd01`**

**Date:** **`09/01/2025`**

![Cap.png](/images/imgs_cap/Cap.png)

---
# Introduction

**_Cap_** is an **Easy-level Linux** machine.

By analyzing a simple **PCAP** file, it is possible to extract sensitive credentials and quickly gain initial access to the system.

The Privilege Escalation phase is achieved by abusing a misconfigured Linux capability that allows code execution as **root**.

---
# Techniques Used

- **IDOR (Insecure Direct Object Reference)**
- **PCAP Analysis** → **Sensitive Data Exfiltration**
- **Abuse of a misconfigured Linux Capability**

---
# Enumeration

## Nmap

Initial full port scan:

```bash
nmap -p- cap -T4
```

![nmap1.png](/images/imgs_cap/nmap1.png)

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -T4 cap
```

![nmap2.png](/images/imgs_cap/nmap2.png)

**Open ports**:
- **21/tcp** → FTP
- **22/tcp** → SSH
- **80/tcp** → HTTP
---
# HTTP - Web Enumeration | http://cap:80

While browsing the web page, I noticed the section **“Security Snapshot (5 Second PCAP + Analysis)”**, which displays information related to captured network packets.

![3.png](/images/imgs_cap/3.png)

There are **two interesting aspects**:
- By clicking the **Download** button, it is possible to download a file containing network packets.
- The content changes by modifying the numeric identifier (ID) in the URL:

This is a clear **IDOR (Insecure Direct Object Reference)** vulnerability: _the application lacks proper authorization checks, allowing anyone to access previous network captures simply by iterating the ID._

![changes0.png](/images/imgs_cap/changes0.png)

So, I downloaded file **“0”** and analyzed it using **Wireshark**.

---
# Initial Access | PCAP Analysis → Sensitive Data Exfiltration

I identified traffic containing **cleartext credentials**:

![creds.png](/images/imgs_cap/creds.png)

**Credentials found**:
`nathan:Buck3tH4TF0RM3!`

I used these credentials to gain **initial access** to the machine via **SSH**:

![ssh_nat.png](/images/imgs_cap/ssh_nat.png)

The **user flag** is located in `/home/nathan`.

---
## Privilege Escalation | Abuse of a Misconfigured Linux Capability

To identify a privilege escalation path, I started enumerating the system locally.

```bash
getcap -r / 2>/dev/null
```

![cap_vuln.png](/images/imgs_cap/cap_vuln.png)

I noticed that the **python3.8** binary had a dangerous capability assigned: **`cap_setuid`**.

As I usually do in these situations, I checked **GTFOBins**:
- https://gtfobins.github.io/gtfobins/python/

By abusing this capability, it is possible to obtain a **root shell**:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash");'
```

![root.png](/images/imgs_cap/root.png)

The **root flag** is located in `/root`.

---
# Final Thoughts

One of the first Linux boxes I ever tackled: very simple, yet excellent for reinforcing the fundamentals of **enumeration**, **network traffic analysis**, and **post-exploitation** in Linux environments.
It clearly demonstrates how seemingly harmless information can become critical if improperly exposed, and how a single misconfiguration can have a significant impact on overall system security.

**Sources**:

- GTFOBins | https://gtfobins.github.io/gtfobins/python/