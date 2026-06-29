+++
date = '2026-06-24T19:25:01+02:00'
draft = false
title = 'Netmon Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`15/02/2025`**

![Netmon.jpeg](/images/imgs_netmon/Netmon.jpeg)

---
# Introduzione

**_Netmon_** è una macchina **Windows** di livello **Easy** che si concentra sullo sfruttamento di un'applicazione di monitoraggio di rete esposta chiamata **PRTG Network Monitor**.

Il percorso di sfruttamento è molto lineare e si basa fortemente su classiche tecniche di enumerazione. Inizia con un **accesso FTP anonimo**, che ci permette di esplorare il file system, recuperare la **User flag** e scoprire vecchi file di backup della configurazione.

Estraendo le credenziali da questi backup e applicando un po' di logica nel **password guessing**, possiamo accedere all'interfaccia web di **PRTG**. Da lì, ottenere una shell è solo questione di sfruttare una nota vulnerabilità di **RCE Autenticata**, che ci garantisce in modo diretto una shell come **NT AUTHORITY\SYSTEM**.

---
# Tecniche Utilizzate
- **Accesso FTP Anonimo**

- **Information Disclosure → File di Backup**

- **RCE Autenticata → Metasploit**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- netmon -T4
```

![nmap_p.png](/images/imgs_netmon/nmap_p.png)

Ho trovato alcune **porte aperte**, ma 3 di queste hanno subito catturato la mia attenzione:
- **21**/tcp - FTP

- **80**/tcp - HTTP

- **445**/tcp - microsoft-ds (SMB)

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV netmon
```

![nmap.png](/images/imgs_netmon/nmap.png)

**Informazioni Utili raccolte**:

1. Il servizio **FTP** consente l'**accesso anonimo**.

2. Il server web ospita "**PRTG Network Monitor**".

3. La versione del servizio **SMB** sembra essere obsoleta.


## FTP - Accesso Anonimo & User Flag

Dato che **FTP** consentiva l'accesso anonimo, è stato il mio primissimo bersaglio. Ho effettuato l'accesso senza fornire alcuna password:

```bash
ftp netmon                                                                       
Connected to netmon.
220 Microsoft FTP Service
Name (netmon:fallingstar): anonymous 
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp>
```

Avendo accesso al file system, ho iniziato a enumerare le directory. Navigando nella cartella **`Users\Public`** sono arrivato dritto alla **user flag**:

![ftp_user.png](/images/imgs_netmon/ftp_user.png)

Dopo aver messo al sicuro la **user flag**, ho spostato la mia attenzione sull'applicazione web. Ho fatto qualche ricerca su Google riguardante **PRTG Network Monitor** per capire come fosse strutturata l'applicazione e dove venissero salvati di default i file di configurazione.

Con queste informazioni, sono tornato alla mia sessione **FTP** attiva e ho navigato nel percorso di configurazione:
**`C:\ProgramData\Paessler\PRTG Network Monitor`**

![config.png](/images/imgs_netmon/config.png)

Qui ho fatto centro. Ho trovato un file di backup chiamato **`PRTG Configuration.old.bak`**.
L'ho scaricato sulla mia macchina locale usando il comando **`get`** e l'ho analizzato.

Cercando parole chiave come **"password"**, ho trovato un set di credenziali:

![admin_creds.png](/images/imgs_netmon/admin_creds.png)

Credenziali recuperate: **prtgadmin:PrTg@dmin2018**

---
## HTTP - Enumerazione Web

Armato di queste credenziali, sono andato su **`http://netmon`** per accedere all'applicazione web **PRTG**.

Inizialmente, il login è fallito. Tuttavia, guardando la password (**PrTg@dmin2018**), era altamente probabile che l'amministratore avesse semplicemente aggiornato la password incrementando l'anno.

Ho cambiato l'anno da **2018** a **2019**, provando:

**prtgadmin:PrTg@dmin2019**

**Nota**: _La scelta del **2019** non è stata un tentativo casuale. Dato che questa macchina è stata rilasciata originariamente su **HackTheBox** nel **2019**, era l'anno più logico da provare per la rotazione della password._

Ha funzionato perfettamente. Una volta dentro, ho controllato la dashboard e ho notato la versione specifica dell'app in esecuzione.

![webapp_logged.png](/images/imgs_netmon/webapp_logged.png)

Conoscendo la versione esatta e avendo accesso come **administrator**, ho aperto **Metasploit** per cercare moduli exploit disponibili.

---
# Accesso Iniziale & Privilege Escalation | PRTG RCE Autenticata

All'interno di **msfconsole**, ho cercato gli exploit per **PRTG** e ho deciso di usare il modulo **`exploit/windows/http/prtg_authenticated_rce`**.

Questo specifico exploit sfrutta una vulnerabilità di **OS command injection** nell'interfaccia web di **PRTG Network Monitor**, permettendo a un utente autenticato di **eseguire comandi arbitrari**.

Ho impostato le opzioni richieste (**RHOSTS, LHOST, LPORT e i cookie/credenziali HTTP recuperati**):

![msfconsole1.png](/images/imgs_netmon/msfconsole1.png)

**Nota**: _Poiché il servizio **PRTG** su questa macchina è in esecuzione con privilegi elevati, sfruttare questa vulnerabilità web non restituisce semplicemente una shell a bassi privilegi, ma ci catapulta direttamente in un contesto amministrativo._

Ho eseguito l'exploit e ho immediatamente ottenuto una reverse shell come:

**_NT AUTHORITY\SYSTEM_**

Avendo il pieno controllo del sistema, ho navigato in **`C:\Users\Administrator\Desktop`** e ho preso la **root flag**:

![root.png](/images/imgs_netmon/root.png)

---
# Conclusioni Finali

**_Netmon_** è una macchina semplice, veloce e molto lineare, nonché un eccellente terreno di pratica per certificazioni come l'**eJPT**.

Mentre la fase iniziale si basa su un classico espediente da **CTF** ovvero lasciare l'intero disco **`C:\`** esposto tramite **FTP anonimo**, la scoperta di vecchi file **`.bak`** è un errore di configurazione ampiamente riconosciuto nel campo della cybersecurity. Funge da ottimo promemoria didattico per ricordarsi di controllare sempre eventuali residui di configurazione e file di backup durante l'enumerazione.

Anche la piccola modifica alla password (passare dal **2018** al **2019**) è un tocco brillante. Simula perfettamente la pigrizia umana quando si tratta di policy di rotazione delle password. Infine, l'uso di **Metasploit** per trasformare un accesso web amministrativo in una shell **SYSTEM** chiude il box in modo eccellente.