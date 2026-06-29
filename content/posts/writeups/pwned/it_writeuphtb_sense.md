+++
date = '2026-06-24T20:47:37+02:00'
draft = false
title = 'Sense Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`04/02/2025`**

![Sense.jpeg](/images/imgs_sense/Sense.jpeg)

---
# Introduzione

**_Sense_** è una macchina **FreeBSD** di livello **Easy** basata su **pfSense**, una distribuzione software open-source per firewall e router ampiamente utilizzata.

Il percorso per compromettere questa macchina si basa fortemente su un'enumerazione approfondita delle directory per scoprire file nascosti. Dopo aver trovato un file di testo contenente uno username, possiamo combinarlo con la password di default del software per accedere alla dashboard amministrativa. Una volta autenticati, la macchina viene compromessa rapidamente sfruttando una nota vulnerabilità di **Command Injection** nell'interfaccia web di **pfSense**, che garantisce comodamente un accesso diretto come **root**.

---
# Tecniche Utilizzate

- **Fuzzing / Enumerazione di Directory**

- **Information Disclosure → system-users.txt**

- **Command Injection Autenticata → RCE**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV -T4 sense
```

![nmap.png](/images/imgs_sense/nmap.png)

**Porte Aperte**:
- **80**/tcp - HTTP (lighttpd 1.4.35)

- **443**/tcp - HTTPS (lighttpd 1.4.35)

## HTTP/HTTPS - Enumerazione Web

Navigando all'URL **`http://sense/`**, il server ci reindirizza a **`https://sense/`** e restituisce il seguente messaggio di errore:

```text
Potential DNS Rebind attack detected, see http://en.wikipedia.org/wiki/DNS_rebinding
Try accessing the router by IP address instead of by hostname. (https://sense/index.php?logout)
```

Seguendo il consiglio dell'errore, ho navigato direttamente verso l'indirizzo IP (**`https://10.10.10.60`**) e mi sono trovato davanti a una pagina di login di **pfSense**.

![pfsense.png](/images/imgs_sense/pfsense.png)

Ho cercato su Google le credenziali di default di **pfSense**, che di solito sono **admin:pfsense**.

Ho provato ad accedere, ma non hanno funzionato.

## Searchsploit

A questo punto, ho deciso di cercare vulnerabilità note per **pfSense**.

```bash
searchsploit pfsense
```

![multiple.png](/images/imgs_sense/multiple.png)

Ho trovato diversi exploit ma, senza conoscere la versione specifica in esecuzione sul target, lanciarli alla cieca non era una buona idea. 

**Nota**: _Considerando l'utilizzo di **HTTPS** e l'avviso di sicurezza riscontrato in precedenza (il blocco per potenziale attacco **DNS Rebind**), c'era la concreta possibilità che fosse attivo un **IDS** (**Intrusion Detection System**) o un meccanismo di ban dell'**IP** pronto a bloccarmi in caso di pattern di attacco evidenti o traffico anomalo. Fare troppo "rumore" in questi scenari è sempre sconsigliato._

Prima di provarne qualcuno, ho deciso di eseguire una scansione con **Gobuster** per enumerare le directory, cercando esplicitamente file **`.txt`**, **`.php`** e **`.html`**. Speravo di trovare maggiori informazioni sulla versione di **pfSense**, configurazioni esposte o, eventualmente, delle credenziali.

## Gobuster

```bash
gobuster dir -u https://10.10.10.60 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k -x txt,php,html
```

Per questa scansione, ho usato la wordlist **`directory-list-2.3-medium.txt`** di **DirBuster**.
Questa lista impiega un po' più di tempo per finire rispetto ad altre, ma nella mia esperienza offre risultati nettamente migliori quando si fa **fuzzing** alla ricerca di estensioni di file specifiche come **`.txt`** o **`.php`** sui server web.

Mentre aspettavo che **Gobuster** finisse, ho deciso di testare manualmente uno degli exploit che avevo trovato prima:

```bash
pfSense < 2.1.4 - 'status_rrd_graph_img.php' Command Injection | php/webapps/43560.py
```

Questo script sfrutta una vulnerabilità di **command injection** in **`status_rrd_graph_img.php`** per iniettare del codice per una reverse shell. Ho provato a eseguirlo più volte, ma ho subito incontrato degli ostacoli.

All'inizio, ha restituito un errore relativo al certificato **SSL**. Ho modificato lo script Python aggiungendo il parametro **`verify=False`** per disabilitare la verifica **SSL** durante la richiesta di login. Tuttavia, anche dopo aver ritoccato il codice, non sono riuscito a farlo funzionare correttamente.

**Nota**: _Sense è una macchina **HackTheBox** molto vecchia. Cercare di eseguire vecchi exploit scritti in **Python 2** su un'installazione moderna di **Kali Linux** (**2025/2026**) porta spesso a conflitti di librerie e fallimenti nella **negoziazione SSL**, anche quando si utilizzano **Virtual Environment Python**._

Nel frattempo, **Gobuster** ha terminato la scansione trovando alcuni file interessanti:

![gob.png](/images/imgs_sense/gob.png)

Ho passato un po' di tempo ad analizzare i risultati e un file in particolare ha catturato la mia attenzione: **`system-users.txt`**.

![sys-users.png](/images/imgs_sense/sys-users.png)

All'interno del file di testo, ho trovato un nome utente: **rohit**.

Sono tornato alla pagina di login di **pfSense** e ho provato ad accedere con il nome utente appena scoperto e la password di default di **pfSense**: **rohit:pfsense**.

Ha funzionato e sono finalmente entrato nella dashboard web. Qui ho subito notato l'esatta versione di **pfSense**: **2.1.3-RELEASE (amd64)**.

---
# Accesso Iniziale & Privilege Escalation | Direct Root Shell

## Metasploit

Leggere la versione **2.1.3** mi ha fatto capire che la vulnerabilità che avevo provato a sfruttare in precedenza (**Command Injection** su **`status_rrd_graph_img.php`**) era effettivamente il percorso corretto.

Tuttavia, invece di testare ulteriori script Python obsoleti, ho deciso di usare **Metasploit**, che ha un modulo integrato per questa esatta **CVE** capace di gestire la **negoziazione SSL** e l'invio del payload senza intoppi.

Ho avviato **msfconsole** e ho caricato il modulo appropriato:

![msf.png](/images/imgs_sense/msf.png)

Dopo aver impostato le opzioni richieste (**RHOSTS, LHOST, LPORT** e le credenziali **rohit:pfsense**), ho eseguito l'exploit.

Poiché l'applicazione web di **pfSense** viene eseguita con i privilegi di sistema più alti per gestire il routing e le regole del firewall, sfruttare questa interfaccia web garantisce una shell **root** diretta.

- La **User flag** si trovava nella home directory di **rohit**: **`/home/rohit/user.txt`**

![userf.png](/images/imgs_sense/userf.png)

- La **Root flag** si trovava in **`/root/root.txt`**

![rootf.png](/images/imgs_sense/rootf.png)

---
# Considerazioni Finali

**_Sense_** è una classica macchina **HTB** che insegna una lezione importantissima: **la pazienza durante l'enumerazione**. Questo la rende un eccellente terreno di pratica per certificazioni come l'**eJPT**.

Affidarsi esclusivamente alle configurazioni di default dei tool potrebbe farvi bloccare. Aggiungere estensioni specifiche come **`.txt`** alle scansioni di **Gobuster** è ciò che fa la differenza per completare questo box. Inoltre, evidenzia l'importanza dell'adattabilità. Di norma preferisco un approccio manuale, in **"stile OSCP"**, allo sfruttamento delle vulnerabilità, senza affidarmi a framework automatizzati. Tuttavia, quando uno script manuale obsoleto fallisce a causa dei limiti di un ambiente moderno, sapere come cambiare rotta e utilizzare in modo efficace un tool come **Metasploit** per raggiungere lo stesso obiettivo è una competenza fondamentale e molto realistica.