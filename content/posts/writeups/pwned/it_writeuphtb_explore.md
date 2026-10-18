+++
date = '2026-10-14T11:14:44+02:00'
draft = false
title = 'Explore Writeup IT'
+++
**Author:** **`joy.scd01`**

**Date:** **`12/10/2026`**

![pwn.jpeg](/images/imgs_explore/pwn.jpeg)

---
# Introduzione

_**Explore**_ è una macchina **Android** di livello **Easy**. Un'ottima introduzione al mondo della **mobile e Android exploitation**.

L'**accesso iniziale** si ottiene sfruttando una vulnerabilità critica in un noto file manager per Android (**ES File Explorer**), il quale espone un server HTTP locale che permette un **Arbitrary File Read (CVE-2019-6447)**. Enumerando e scaricando file sensibili dalla memoria del dispositivo, è possibile recuperare delle **credenziali SSH** valide nascoste all'interno di un'immagine.

La **privilege escalation** prevede il **local port forwarding** per interagire con un servizio interno **Android Debug Bridge (ADB)** in ascolto sulla porta **`5555`**, che può essere facilmente abusato per ottenere una **root shell** tramite il comando **`su`**.

---
# Tecniche Utilizzate

- **ES File Explorer Arbitrary File Read (CVE-2019-6447)**
- **Information Disclosure (Credenziali in un'immagine) → Accesso SSH**
- **SSH Port Forwarding**
- **ADB (Android Debug Bridge) Exploitation → Root**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV explore
```

![nmap1.png](/images/imgs_explore/nmap1.png)

Scansione completa di tutte le porte per identificare ulteriori servizi attivi su porte non standard:

```bash
nmap -p- explore
```

![nmap2.png](/images/imgs_explore/nmap2.png)

**Porte aperte**:

- **2222**/tcp SSH (Banana Studio SSH Server)

- **42135**/tcp unknown

- **59777**/tcp unknown

## Enumerazione dei Servizi (Porte Android)

Inizialmente ho fatto qualche ricerca su **Banana Studio SSH**.

Si tratta di un'applicazione di terze parti progettata per eseguire un **server SSH** su dispositivi **Android** (spesso utilizzata su **Android TV** o emulatori per la gestione remota). Poiché viene eseguita in user-space senza privilegi di **root** di default, si mette in ascolto sulla porta non privilegiata **`2222`** invece della porta standard **`22`**. Non avendo exploit pubblici rilevanti, servivano delle credenziali valide per effettuare l'accesso.

Successivamente, mi sono concentrato sulle altre due porte: **`42135`** e **`59777`**.
Cercando online informazioni sui servizi **Android** associati alla porta **`59777`**, ho scoperto che è direttamente collegata all'applicazione **ES File Explorer**, vulnerabile alla **CVE-2019-6447**.

**Nota**: _Quando **ES File Explorer** (nelle versioni precedenti alla **`4.1.9.7.4`**) viene avviato su un dispositivo **Android**, fa partire automaticamente un server **HTTP** nascosto e privo di autenticazione in ascolto su **`0.0.0.0:59777`**. Qualsiasi attaccante sulla stessa rete può inviare richieste **JSON POST** a questa porta per eseguire comandi amministrativi, **elencare file o immagini**, **recuperare informazioni sul dispositivo** e **scaricare file arbitrari** dalla memoria._

---
# Initial Access | CVE-2019-6447 → Arbitrary File Read → kristi

Per sfruttare questa vulnerabilità, ho utilizzato il seguente **PoC** pubblico presente su **Exploit-DB**:
- https://www.exploit-db.com/exploits/50070

Dopo aver analizzato e scaricato lo script, ho iniziato ad enumerare il dispositivo eseguendo i comandi integrati nell'exploit:

```bash
python3 50070.py getDeviceInfo 10.129.238.59
python3 50070.py listFiles 10.129.238.59
python3 50070.py listPics 10.129.238.59
```

![exp1.png](/images/imgs_explore/exp1.png)

![exp2.png](/images/imgs_explore/exp2.png)

Elencando le immagini tramite il modulo **`listPics`**, ho notato qualcosa di molto interessante:

```text
name : creds.jpg
time : 4/21/21 02:38:18 AM
location : /storage/emulated/0/DCIM/creds.jpg
size : 1.14 MB (1,200,401 Bytes)
```

![exp3.png](/images/imgs_explore/exp3.png)

Ho quindi usato il modulo **`getFile`** per scaricare **`creds.jpg`** sulla mia macchina locale:

```bash
python3 50070.py getFile 10.129.238.59 /storage/emulated/0/DCIM/creds.jpg
```

L'immagine conteneva le credenziali per l'utente **`kristi`**, scritte su un foglietto di carta.

![creds.png](/images/imgs_explore/creds.png)

Ho utilizzato queste credenziali per connettermi al server **Banana Studio SSH** sulla porta **`2222`**:

```bash
ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa kristi@explore -p 2222
```

**Nota**: _Poiché si tratta di una macchina datata e la sto pwnando nel **2026** con una distribuzione **Kali Linux** aggiornata, i moderni client **OpenSSH** disabilitano di default gli algoritmi crittografici legacy. Passare i parametri **`-o KexAlgorithms=+diffie-hellman-group14-sha1`** e **`-o HostKeyAlgorithms=+ssh-rsa`** è necessario per evitare errori di negoziazione delle chiavi con il vecchio **server SSH Android**._

![initial.png](/images/imgs_explore/initial.png)

---
# Privilege Escalation | SSH Port Forwarding → ADB (Port 5555) → root

Una volta dentro la macchina, non avendo familiarità con la struttura standard delle directory di **Android**, ho iniziato un'enumerazione manuale alla ricerca di vettori per fare privilege escalation.

Controllando le connessioni di rete interne attive con **`ss`**:

```bash
ss -tuln
```

![5555.png](/images/imgs_explore/5555.png)

Ho scoperto un servizio interno in ascolto localmente sulla porta **`5555`**.

**Nota**: _La porta **`5555`** è la porta **TCP/IP** predefinita utilizzata dal demone **Android Debug Bridge** (**ADB**, ovvero **adbd**) sui dispositivi ed emulatori **Android** per consentire il debugging tramite rete._

Poiché la porta **`5555`** era in ascolto solo su localhost, ho impostato un tunnel tramite **SSH Local Port Forwarding** per inoltrare la porta remota **`5555`** sulla mia macchina **Kali** locale:

```bash
ssh -L 5555:localhost:5555 -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa kristi@explore -p 2222
```

Da un altro terminale sulla mia macchina, ho utilizzato il tool **`adb`** per connettermi alla porta inoltrata e avviare una shell interattiva:

```bash
adb connect localhost:5555
adb shell
```

Da qui ho provato semplicemente ad eseguire **`su`**:

```bash
su
```

Che mi ha garantito una root shell (**root flag** situata in **`/data/root.txt`**).

![rootf.png](/images/imgs_explore/rootf.png)

**Perché ha funzionato così facilmente?**

_Mentre la sessione SSH come **`kristi`** gira all'interno dell'ambiente sandboxed dell'applicazione di terze parti **Banana Studio** (che non ha i permessi per interagire con il binario **`su`**), la connessione tramite **ADB** ci assegna l'utente privilegiato di sistema **`shell`** (**`uid=2000`**). Poiché questa macchina esegue una build di **Android** con permessi di root/sviluppatore (build **`userdebug`** o **`eng`**), il binario **`su`** è configurato per consentire all'utente **`shell`** di scalare i privilegi a **root** senza richiedere una password o una conferma tramite interfaccia grafica._

**Ora la parte difficile**

Con i privilegi di **root** finalmente in mio possesso, dovevo ancora trovare la **user flag**. Dopo aver vagato per un po' nell'albero delle directory di **Android**, l'ho trovata all'interno della memoria condivisa emulata in **`/storage/emulated/0/user.txt`**:

![userf.png](/images/imgs_explore/userf.png)

---
# Considerazioni Finali

Un'ottima introduzione al mondo dell'**Android exploitation**.

Una macchina super semplice, ma molto istruttiva per prendere confidenza con strumenti come **`adb`**, capire come funziona l'**Android Debug Bridge** (**`5555`**) e imparare a muoversi nella struttura del file system **Android**. Onestamente, forse la parte più difficile di tutta la macchina è stata capire dove fosse nascosta la **user flag**! :)

**Fonti**:

- **CVE-2019-6447 Exploit (ES File Explorer) | https://www.exploit-db.com/exploits/50070**

- **Android Debug Bridge (ADB) Documentation | https://developer.android.com/tools/adb**