+++
date = '2026-05-19T21:10:36+02:00'
draft = false
title = 'Beep Writeup IT'
+++
**Nome:** `joy.scd01`

**Data:** `03/02/2025`

![Beep.png](/images/imgs_beep/Beep.png)

---
# Introduzione

**_Beep_** è una macchina **Linux** di difficoltà **Easy**, pensata per esercitarsi con diversi approcci di **penetration testing** di base.
Personalmente l’ho rootata tramite una vulnerabilità di **Local File Inclusion**, ma esistono diversi percorsi di attacco validi, tra cui:

- **LFI tramite graph.php**
- **CVE-2012-4867 | vTiger CRM 5.1.0 LFI**
- **Webmin 1.570 RCE (Metasploit)**
- **CVE-2012-4869**

---
# Enumerazione

## Nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- beep
```

![nmap1.png](/images/imgs_beep/nmap1.png)

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -T4 beep
```

![nmap2.png](/images/imgs_beep/nmap2.png)

**Punti principali di interesse**:

- **80**/tcp http

- **443**/tcp ssl/http

---
## HTTP - Enumerazione Web | http://beep:80

La pagina web hosta un’interfaccia login di **Elastix**.
Per qualche motivo ho dovuto modificare la variabile **`security.tls.version`** in **Firefox** per riuscire a visualizzarla.

Poiché l’applicazione non mostrava informazioni utili sulla versione, ho cercato direttamente **elastix** su **Searchsploit**:

![searc.png](/images/imgs_beep/searc.png)

Dato che la vulnerabilità di **XSS** richiedeva un utente autenticato, mi sono concentrato sulla **LFI in `graph.php`**.

_**`graph.php`** è vulnerabile ad una **Local File Inclusion** perché non sanitizza correttamente l’input dell’utente.
Attraverso payload con path traversal è possibile recuperare **file locali** ed eventualmente **eseguire script** sul server._

---
# Accesso Iniziale |  LFI tramite graph.php

Ho provato il payload suggerito:

```text
/vtigercrm/graph.php?current_language=../../../../../../../../etc/amportal.conf%00&module=Account&action
```

![lfi.png](/images/imgs_beep/lfi.png)

**(Aprire il sorgente della pagina per capire meglio)** :

![creds.png](/images/imgs_beep/creds.png)

Il file recuperato conteneva delle **credenziali**.

Ho usato l’LFI anche per leggere il file **`/etc/passwd`**, dal quale ho estratto gli utenti in una userlist e ho provato un bruteforce con **Hydra**.
Sembra però esserci una protezione contro questo tipo di attacco.
Quindi, dato che utenti e password erano pochi, li ho testati **manualmente**.

---
## SSH Issue

Provando le credenziali via SSH, ho riscontrato un problema con gli **algoritmi di key exchange**:

![algoerror.png](/images/imgs_beep/algoerror.png)

**Nota**:
_Questo succede perché **Beep** è una macchina molto vecchia ed io la sto facendo nel **2025** usando una versione moderna di **Kali Linux Purple**.
Il server SSH supporta solo algoritmi obsoleti che ormai non sono più abilitati di default nei client moderni.
**Per risolvere**, è necessario specificare manualmente gli algoritmi più vecchi compatibili con il server._

---

La password **`jEhdIekWmdjE`** era valida per l’utente **root**.
Quindi, una volta connesso, ho ottenuto direttamente il controllo totale della macchina:

```bash
ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa root@beep
```

![root.png](/images/imgs_beep/root.png)

Qui ho potuto recuperare **entrambe le flag**.

---
# Conclusioni Finali

_Beep_ è un box ideale per chi vuole un primo approccio al **penetration testing**.
I vettori di attacco sono semplici ma numerosi, e questo offre l’opportunità di mettere in pratica tecniche diverse.
La macchina aiuta ad entrare nell’ottica reale del pentesting: spesso non esiste un’unica strada e valutando la sicurezza dei vari servizi esposti si possono trovare più modi per compromettere un sistema.