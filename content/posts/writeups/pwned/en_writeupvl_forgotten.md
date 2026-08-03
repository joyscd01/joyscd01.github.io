+++
date = '2026-05-23T18:26:30+02:00'
draft = false
title = 'Forgotten Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`23/06/2025`**

![forgotten_slide.png](/images/imgs_forgotten/forgotten_slide.png)

---
# Introduction

**Forgotten** is an _Easy_-level Linux machine, interesting due to its initial setup requiring a **local MySQL container**, which then connects to an exposed **LimeSurvey** instance.
From there, by uploading a **malicious plugin**, I gained initial access to the box.

Later, a **password reuse** allowed me to laterally move from the container to the host. Finally, by abusing permissions and shared directories between the two environments, I performed a **container breakout**, gaining **root** access.

---
# Techniques Used

- **Malicious plugin upload for Remote Code Execution**
- **Password reuse**
- **Container breakout via SUID permissions**

---
# Enumeration

## nmap

Basic script and service scan:

```bash
nmap -sC -sV forgotten
```

![nmap.png](/images/imgs_forgotten/nmap.png)

**Open ports**:
- **22/tcp** - SSH
- **80/tcp** - HTTP

## HTTP - Web enumeration

Visiting the website, I was greeted with a generic "Forbidden" message:

![web-server.png](/images/imgs_forgotten/web-server.png)

I ran **gobuster** to look for hidden directories or files.

```bash
gobuster dir -u http://forgotten/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gobuster.png](/images/imgs_forgotten/gobuster.png)

Among the results, I found a **LimeSurvey installer** and its **version**.

![installer.png](/images/imgs_forgotten/installer.png)

---
# Initial Access | LimeSurvey Setup → Malicious Plugin Upload → RCE

**Note**: _**LimeSurvey** is an open-source survey management platform._

Following the installation steps, it asked for **database connection details**.

So, using Docker, I quickly set up a **local MySQL container**.
To do this, I literally Googled: "**how to create a local mysql database with docker**".
The browser **AI** pretty much gave me the exact command and explained the flags. I just added the required parameters from the form — _super intuitive_.

```bash
sudo docker run --name limesurvey_db -e MYSQL_ROOT_PASSWORD=pentest -e MYSQL_DATABASE=limesurvey -e MYSQL_USER=limesurvey_user -e MYSQL_PASSWORD=pentest -p 3306:3306 -d mysql:latest
```

![db.png](/images/imgs_forgotten/db.png)

I filled in the setup form with the credentials and completed the installation.

![installer2.png](/images/imgs_forgotten/installer2.png)

Once done, I accessed the admin panel using the default credentials `admin` / `password`.

From the plugin panel, I uploaded a **malicious plugin** containing a **PHP reverse shell**.
Uploading, installing, and activating the plugin gave me a shell as the **`limesvc`** user.

**Note**: _I had previously tested LimeSurvey, so recognizing the version, I went straight for the plugin exploit._

Still, searching "**LimeSurvey 6.3.7 CVE**" gives you everything you need.

For the exploit, I used this repo and followed the instructions:

- [https://github.com/Y1LD1R1M-1337/Limesurvey-RCE](https://github.com/Y1LD1R1M-1337/Limesurvey-RCE)

After cloning the repo, I edited the **`php-rev.php`** file to insert my **IP** and **port**:

![php-rev.png](/images/imgs_forgotten/php-rev.png)

I added the version `6.3.7` in **`config.xml`**:

![config.png](/images/imgs_forgotten/config.png)

Then I zipped the plugin:

![zip.png](/images/imgs_forgotten/zip.png)

Uploaded, installed, and activated it:

![plugin.png](/images/imgs_forgotten/plugin.png)

Finally, with a **netcat listener running**, I browsed to:

- [http://forgotten/survey/upload/plugins/Y1LD1R1M/php-rev.php](http://forgotten/upload/plugins/Y1LD1R1M/)

To trigger the **reverse shell**:

![initial-access.png](/images/imgs_forgotten/initial-access.png)

Running **`ifconfig`**, I immediately noticed a suspicious _Hostname_ and _IP_ — classic container signs.

**Note**: _A clear sign of being inside a Docker container is the presence of interfaces like `eth0` with private bridge network IPs. Also, hostnames are usually minimal or default._

At this point, I stabilized the shell.
**Python** wasn’t available, so I used this instead (from a TTY cheat sheet):

Source: [https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet](https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet)

```bash
script -qc /bin/bash /dev/null
```

---
# Lateral Movement | Password Reuse → limesvc

As usual, I began basic enumeration.
Using **`env`**, I found environment variables — one of which leaked the password for **limesvc**.

![env.png](/images/imgs_forgotten/env.png)

I then connected to the **main host** over SSH using those credentials:

![ssh-access.png](/images/imgs_forgotten/ssh-access.png)

Inside the home directory, I found the **user flag**.

---
# Privilege Escalation | Container Breakout → SUID Permissions → root

This is where I hit a wall.
I was tired and seeing double… so I went for some **cross-enumeration between container and host**.
.
.
![maxresdefault.jpg](/images/imgs_forgotten/maxresdefault.jpg)

(_It could’ve been much worse_).

**Here’s what I discovered**:

- Running `sudo -l` **inside the container** showed that the user **limesvc** could run any command as **root**, so I switched to **root** inside the container.

![sudo-l.png](/images/imgs_forgotten/sudo-l.png)

-  Listing files in **`/opt`**, **from the host**, I found a writable directory: **`limesurvey`**.

So, **from the host**, I copied **`/usr/bin/bash`** to **`/opt/limesurvey`**.

**Inside the container**, I navigated to the mapped path:

```bash
cd /var/www/html/survey
```

Then changed the ownership and SUID permission of the **`bash`** binary:

```bash
chown root:root bash
chmod u+s bash
```

Finally, **back on the host**, I executed the binary with preserved root privileges:

```bash
./bash -p
```

![root-proof.png](/images/imgs_forgotten/root-proof.png)

**Root Flag**.

To maintain access, I added my **public SSH key** to `/root/.ssh/authorized_keys`.

---
# Final Thoughts

As usual, I find Vulnlab machines to be **absolute gems**.
They always introduce you to new techniques, alternative tools, and real-world applicable scenarios.

I definitely recommend this box.
The privilege escalation was slightly tricky for an _Easy_ box, but it was very satisfying.
It helped me solidify the **container breakout technique**, and better understand the risks of poorly configured container environments.

I noticed some guides use tools to analyze Docker containers and automate this process — but in my opinion, this part was **more rewarding when done manually**, and that’s the approach I stick to, especially while prepping for **OSCP**.

Also, I appreciated the quick hands-on with **Docker** for setting up a local **MySQL instance**, which was key to getting in.

**Resources**:

-  **Malicious plugin exploit repo for LimeSurvey** | https://github.com/Y1LD1R1M-1337/Limesurvey-RCE
-  **Shell stabilization cheat sheet** | [https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet](https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet)