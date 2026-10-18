+++
date = '2026-10-14T11:14:41+02:00'
draft = false
title = 'Explore Writeup EN'
+++
**Author:** **`joy.scd01`**

**Date:** **`12/10/2026`**

![pwn.jpeg](/images/imgs_explore/pwn.jpeg)

---
# Introduction

_**Explore**_ is an **Easy-level Android machine** that serves as a great introduction to the world of **mobile and Android exploitation**.

The **initial access** is achieved by abusing a critical vulnerability in a well-known Android file manager (**ES File Explorer**), which exposes an undocumented local HTTP server allowing **Arbitrary File Read (CVE-2019-6447)**. By enumerating and downloading sensitive files from the device's storage, we can recover valid **SSH credentials** hidden inside an image.

The **privilege escalation** involves **local port forwarding** to interact with an internal **Android Debug Bridge (ADB)** service listening on port **`5555`**, which can be easily abused to spawn a **root shell** via **`su`**.

---
# Techniques Used

- **ES File Explorer Arbitrary File Read (CVE-2019-6447)**
- **Information Disclosure (Credentials in Image) → SSH Access**
- **SSH Port Forwarding**
- **ADB (Android Debug Bridge) Exploitation → Root**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV explore
```

![nmap1.png](/images/imgs_explore/nmap1.png)

Full port scan to identify additional running services on non-standard ports:

```bash
nmap -p- explore
```

![nmap2.png](/images/imgs_explore/nmap2.png)

**Open ports**:

- **2222**/tcp SSH (Banana Studio SSH Server)

- **42135**/tcp unknown

- **59777**/tcp unknown

## Service Enumeration (Android Ports)

At first, I researched **Banana Studio SSH**.

It is a lightweight third-party application designed to run an SSH server on Android devices (often used on Android TVs or emulators for remote management). Since it runs in user-space without root privileges by default, it binds to the non-privileged port **`2222`** instead of the standard port **`22`**. It did not have any relevant public exploits, meaning we needed valid credentials to log in.

Next, I focused on the two unknown high ports: **`42135`** and **`59777`**.
By searching online for Android services associated with port **`59777`**, I discovered that it is directly linked to the **ES File Explorer** application, which is vulnerable to **CVE-2019-6447**.

**Note**: _When **ES File Explorer** (versions prior to **`4.1.9.7.4`**) is launched on an **Android** device, it automatically starts a hidden, unauthenticated **HTTP** server bound to **`0.0.0.0:59777`**. Any attacker on the same network can send **JSON POST** requests to this port to execute administrative commands, **list files/pictures**, **retrieve device info**, and **download arbitrary files** from the device's storage._

---
# Initial Access | CVE-2019-6447 → Arbitrary File Read → kristi

To exploit this vulnerability, I used the following public **PoC** from **Exploit-DB**:
- https://www.exploit-db.com/exploits/50070

After analyzing and downloading the script, I started enumerating the device by running its built-in commands:

```Bash
python3 50070.py getDeviceInfo 10.129.238.59
python3 50070.py listFiles 10.129.238.59
python3 50070.py listPics 10.129.238.59
```

![exp1.png](/images/imgs_explore/exp1.png)

![exp2.png](/images/imgs_explore/exp2.png)

While listing the pictures with **`listPics`**, I noticed something very interesting:

```text
name : creds.jpg
time : 4/21/21 02:38:18 AM
location : /storage/emulated/0/DCIM/creds.jpg
size : 1.14 MB (1,200,401 Bytes)
```

![exp3.png](/images/imgs_explore/exp3.png)

I used the **`getFile`** module to download **`creds.jpg`** to my local machine:

```bash
python3 50070.py getFile 10.129.238.59 /storage/emulated/0/DCIM/creds.jpg
```

The image contained the credentials for the user **`kristi`**, written on a piece of paper.

![creds.png](/images/imgs_explore/creds.png)

I then used these credentials to connect to the **Banana Studio SSH** server on port **`2222`**:

```bash
ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa kristi@explore -p 2222
```

**Note**: _Since this is an older box and I am tackling it in **2026** using an updated **Kali Linux** distribution, the modern **OpenSSH** client disables legacy cryptographic algorithms by default. Passing **`-o KexAlgorithms=+diffie-hellman-group14-sha1`** and **`-o HostKeyAlgorithms=+ssh-rsa`** is necessary to avoid key negotiation errors with the outdated **Android SSH server**._

![initial.png](/images/imgs_explore/initial.png)

# Privilege Escalation | SSH Port Forwarding → ADB (Port 5555) → root

Once inside the box, I was not familiar with the standard **Android** directory structure, so I started a manual enumeration to look for privilege escalation vectors.
By checking the active internal network connections with **`ss`**:

```bash
ss -tuln
```

![5555.png](/images/imgs_explore/5555.png)

I discovered an internal service listening locally on port **`5555`**:

**Note**: _Port **`5555`** is the default **TCP/IP** port used by the **Android Debug Bridge** (**ADB**) daemon (**adbd**) on **Android** devices and emulators to allow debugging over the network._

Since port **`5555`** was only bound to localhost, I set up an **SSH Local Port Forwarding** tunnel to forward the remote port **`5555`** to my local **Kali** machine:

```bash
ssh -L 5555:localhost:5555 -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa kristi@explore -p 2222
```

From another terminal on my local machine, I used the **`adb`** tool to connect to the forwarded port and spawn an interactive shell:

```bash
adb connect localhost:5555
adb shell
```

At this point, I was dropped into a shell as the shell user. After experimenting for a bit, I simply tried executing **`su`**:

```bash
su
```

And it immediately granted me a root shell (**root flag** inside **`/data/root.txt`**).

![rootf.png](/images/imgs_explore/rootf.png)

**Why did this work so easily?**

_While the SSH session as **`kristi`** runs inside the sandboxed environment of the third-party **Banana Studio** app (which lacks permissions to interact with the **`su`** binary), connecting via **ADB** drops us into the system's privileged shell user (**`uid=2000`**). Because this machine runs a rooted/developer build of Android (userdebug or eng build), the **`su`** binary is configured to allow the shell user to escalate to **root** without requiring a password or a GUI prompt._

**Now the difficult part**

With **root** privileges in hand, I still needed to track down the **user flag**. After wandering around the unfamiliar **Android** directory tree for a while, I eventually found it located inside the shared emulated storage at **`/storage/emulated/0/user.txt`**:

![userf.png](/images/imgs_explore/userf.png)

# Final Thoughts

A great introduction to the **Android exploitation** world.

It is a super easy box, but very educational to get familiar with tools like **`adb`**, understand how the **Android Debug Bridge** (**`5555`**) works, and learn how to navigate the **Android** file system structure. Honestly, maybe the hardest part of the whole box was figuring out where the **user flag** was hidden! :)

**Sources**:

- **CVE-2019-6447 Exploit (ES File Explorer) | https://www.exploit-db.com/exploits/50070**

- **Android Debug Bridge (ADB) Documentation | https://developer.android.com/tools/adb**