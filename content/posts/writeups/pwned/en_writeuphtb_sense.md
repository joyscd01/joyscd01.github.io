+++
date = '2026-06-24T20:47:34+02:00'
draft = false
title = 'Sense Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`04/02/2025`**

![Sense.jpeg](/images/imgs_sense/Sense.jpeg)

---
# Introduction

**_Sense_** is an **Easy-level FreeBSD** machine based on **pfSense**, a widely used open-source firewall and router software distribution.

The path to compromising this box relies heavily on thorough directory enumeration to uncover hidden files. After discovering a custom text file containing a username, we can combine it with the software's default password to access the administrative dashboard. Once authenticated, the machine is quickly owned by exploiting a known **Command Injection** vulnerability in the **pfSense** web interface, which conveniently provides direct **root** access.

---
# Techniques Used
- **Directory Fuzzing / Enumeration**

- **Information Disclosure → system-users.txt**

- **Authenticated Command Injection → RCE**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -T4 sense
```

![nmap.png](/images/imgs_sense/nmap.png)

**Open Ports**:
- **80**/tcp - HTTP (lighttpd 1.4.35)
- **443**/tcp - HTTPS (lighttpd 1.4.35)

---
# HTTP/HTTPS - Web Enumeration

If we browse to the URL **`http://sense/`**, the server redirects us to **`https://sense/`** and throws the following error message:

```text
Potential DNS Rebind attack detected, see http://en.wikipedia.org/wiki/DNS_rebinding
Try accessing the router by IP address instead of by hostname. (https://sense/index.php?logout)
```

Following the error's advice, I navigated to the IP address directly ([https://10.10.10.60](https://10.10.10.60)) and was greeted by a **pfSense** login page.

![pfsense.png](/images/imgs_sense/pfsense.png)

I searched on Google for **pfSense** default credentials, which are usually **admin:pfsense**.

I tried to log in, but they didn't work.

## Searchsploit

At this point, I decided to check for known vulnerabilities in **pfSense**.

```bash
searchsploit pfsense
```

![multiple.png](/images/imgs_sense/multiple.png)

I found multiple exploits, but without knowing the specific version running on the target, blindly firing them off wasn't a good idea.

**Note**: _Considering the use of **HTTPS** and the security warning encountered earlier (the potential **DNS Rebind attack** block), there was a real possibility of an active **IDS** (**Intrusion Detection System**) or **IP** banning mechanism ready to block me if it detected obvious attack patterns or anomalous traffic. Making too much "noise" in these scenarios is never recommended._

 Before trying any of those, I decided to run a **Gobuster** scan to enumerate directories, explicitly searching for **`.txt`**, **`.php`**, and **`.html`** files. I was hoping to find more information about the **pfSense** version, exposed configurations, or eventually, some credentials.

## Gobuster

```bash
gobuster dir -u https://10.10.10.60 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k -x txt,php,html
```

For this scan, I used the **`directory-list-2.3-medium.txt`** wordlist from **DirBuster**.
This list takes a bit longer to finish compared to others (about 30 minutes), but in my experience, it yields much better results when **fuzzing** for specific file extensions like **`.txt`** or **`.php`** on web servers.

While waiting for **Gobuster** to finish, I decided to manually test one of the exploits I had found earlier:
```bash
pfSense < 2.1.4 - 'status_rrd_graph_img.php' Command Injection | php/webapps/43560.py
```

This script exploits a **command injection** vulnerability in **`status_rrd_graph_img.php`** to inject code for a reverse shell. I tried to run it multiple times, but I immediately hit some roadblocks.

At first, it threw an **SSL certificate error**. I modified the Python script by adding the parameter **`verify=False`** to disable **SSL** verification during the login request. However, even after tweaking the code, I wasn't able to make it work properly.

**Note**: _Sense is a very old **HackTheBox** machine. Trying to run ancient **Python 2** exploit scripts on a modern **Kali Linux** installation (**2025/2026**) often leads to **library conflicts** and **SSL negotiation failures**, even when using **Python virtual environments**._

Fortunately, in the meantime, **Gobuster** finished its scan and found some interesting files:

![gob.png](/images/imgs_sense/gob.png)

I spent some time analyzing the results, and one file stood out from the rest: **`system-users.txt`**.

![sys-users.png](/images/imgs_sense/sys-users.png)

Inside the text file, I found a custom username: **rohit**.

I went back to the **pfSense** login page and tried logging in with the newly discovered username and the default **pfSense** password: **rohit:pfsense**.

It worked and I was finally inside the web dashboard. Here, I immediately spotted the exact **pfSense version**: **2.1.3-RELEASE (amd64)**.

This information confirmed that I was on the right track all along to gain my foothold.

---
# Initial Access & Privilege Escalation | Direct Root Shell

## Metasploit

Reading the version **2.1.3** made me realize that the vulnerability I tried to exploit earlier (**`status_rrd_graph_img.php`** **Command Injection**) was indeed the correct path.

However, instead of fighting with outdated Python scripts, I decided to use **Metasploit**, which has a built-in module for this exact CVE that handles the **SSL negotiation** and payload delivery smoothly.

I launched **msfconsole** and loaded the appropriate module:

![msf.png](/images/imgs_sense/msf.png)

After setting up the required options (**RHOSTS, LHOST, LPORT**, and the **rohit:pfsense** credentials), I ran the exploit.

Because the **pfSense** web application runs with the highest system privileges to manage routing and firewall rules, exploiting this web interface grants a direct **root** shell.

Both flags were easily retrievable:

- The **User flag** was located in **rohit**'s home directory: **`/home/rohit/user.txt`**

![userf.png](/images/imgs_sense/userf.png)

- The **Root flag** was located in **`/root/root.txt`**

![rootf.png](/images/imgs_sense/rootf.png)

---
# Final Thoughts

**_Sense_** is a classic **HTB machine** that teaches a very important lesson: **patience during enumeration**. This makes it an excellent practice ground for certifications like the **eJPT**.

Relying solely on default tool configurations might leave you stuck. Appending specific extensions like **`.txt`** to your **Gobuster** scans is what makes or breaks this box. Furthermore, it highlights the importance of adaptability. I normally prefer a manual, **"OSCP-style"** approach to exploitation without relying on automated frameworks. However, when an outdated manual script fails due to modern environment constraints, knowing how to pivot and effectively use a tool like **Metasploit** to achieve the same goal is a crucial and realistic skill.