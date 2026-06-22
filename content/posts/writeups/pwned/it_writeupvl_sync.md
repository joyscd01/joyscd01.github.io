+++
date = '2026-05-21T16:28:36+02:00'
draft = false
title = 'Sync Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`05/06/2025`**

![sync_slide.png](/images/imgs_sync/sync_slide.png)

---
# Introduzione

_Sync_ è una macchina Linux di livello _Easy_.
Questo box è molto interessante per la **catena di escalation progressiva**, che parte da una semplice esposizione del protocollo **rsync**, fino alla compromissione totale del sistema.

L'accesso iniziale è stato ottenuto sfruttando una directory rsync accessibile pubblicamente, dalla quale ho scaricato il database contenente password hashate. Dopo averne fatto il cracking offline, ho avuto accesso iniziale via **FTP**.

Da lì, una catena di **lateral movement** mi ha permesso di spostarmi tra i vari utenti fino a sfruttare un cronjob mal configurato per ottenere i privilegi di **root**.

---
# Tecniche Utilizzate

-  **Password Hash Extraction & Cracking**
-  **Password Reuse**
-  **Unshadow Attack**
-  **Cronjob Abuse**

---
# Enumerazione

**nmap**

Scansione mirata con script e servizi:

```bash
nmap -sC -sV sync
```

![nmap.png](/images/imgs_sync/nmap.png)

**Porte aperte**:
-  **21/tcp** - FTP
-  **22/tcp** - SSH
-  **80/tcp** - HTTP
-  **873/tcp** - RSYNC

**FTP & SSH (porte 21 e 22)**

Come prima cosa ho provato a loggarmi tramite **FTP e SSH**, ma entrambi non supportano il login anonimo.
Così ho aperto il browser per enumerare il servizio **HTTP**.

**HTTP - Web enumeration (porta 80)**

Il servizio hostato sulla **porta 80** espone un semplice login form.
Ho fatto qualche prova per testare una possibile **SQL Injection** e sono riuscito a bypassare il form utilizzando il classico payload `' OR 1=1 -- -` nel parametro "_Username_".

![sqlinjection.png](/images/imgs_sync/sqlinjection.png)

![logged.png](/images/imgs_sync/logged.png)

Non trovando nulla di utile sul sito web, sono passato all'enumerazione del protocollo **rsync**.

**rsync (porta 873)**

Non avevo mai avuto a che fare con questo protocollo, quindi ho fatto una rapida ricerca su Google per capire come interagirci.

Grazie a questo articolo: https://hackviser.com/tactics/pentesting/services/rsync sono riuscito a familiarizzare con i comandi utili per l'enumerazione.

```bash
rsync sync::
```

Il protocollo espone un **modulo accessibile pubblicamente** chiamato `httpd`, al cui interno era presente il **database** del sito (`site.db`).

Ho quindi effettuato il download della directory con il comando per _data exfiltration_:

```bash
rsync -avz sync::httpd httpd
```

---
# Accesso Iniziale | Password Cracking → FTP accesso come triss

Analizzando il database, all'interno della tabella _users_ ho trovato le **password hashate** degli utenti **admin** e **triss**

```sql
sqlite3 site.db
.tables
SELECT * FROM users;
```

![db-hashes.png](/images/imgs_sync/db-hashes.png)

All'interno di `/www`, era presente anche il file `index.php` da cui ho estratto informazioni utili per capire il metodo di hashing.

```bash
cat index.php
```

![crypt.png](/images/imgs_sync/crypt.png)

In particolare:
-  La variabile `$secure` mostra il **salt** che viene utilizzato.
-  La variabile `$hash` mostra il metodo di hashing (**MD5 con concatenazione**)

Ho quindi creato un file `hash.txt` con l'hash dell'utente **triss** seguito dal metodo di hashing:

```text
a0de4d7f81676c3ea9eabcadfd2536f6:6c4972f3717a5e881e282ad3105de01e|triss|
```

A questo punto ho avviato l'attacco con **hashcat**:

```bash
hashcat -a 0 -m 20 hash.txt /usr/share/wordlists/rockyou.txt
```

![cracked.png](/images/imgs_sync/cracked.png)

Siccome all'inizio avevo notato che **SSH richiedeva l'accesso con chiave**, ho provato ad accedere tramite **FTP** ottenendo così l'accesso iniziale:

```bash
ftp triss@sync
```

![ftp-proof.png](/images/imgs_sync/ftp-proof.png)

Ora però, per avere un ambiente pienamente interattivo e per un accesso persistente, ho creato una cartella `.ssh` nella home dell'utente e al suo interno ho creato il file `authorized_keys` contenente la mia chiave RSA pubblica.

```ftp
mkdir .ssh
cd .ssh
put authorized_keys
```

Fatto ciò e configurati i permessi, mi sono loggato tramite:

```bash
ssh -i id_rsa triss@sync
```

![initial-access-proof.png](/images/imgs_sync/initial-access-proof.png)

---
# Lateral Movement 1 - da "triss" a "jennifer" | Password reuse → jennifer

Una volta ottenuto l'accesso, enumerando la macchina, ho notato la presenza di altri 2 utenti: **jennifer** e **sa**.

Quando si hanno delle credenziali e più utenti, è buona cosa provare a **riutilizzare la password**.

**Nota**: _Il riutilizzo di password tra più utenti è una cattiva pratica molto diffusa in ambienti reali. In questo caso, ha permesso uno spostamento laterale senza ulteriori exploit_.

```bash
su jennifer
```

![lateral-proof.png](/images/imgs_sync/lateral-proof.png)

Qui ho trovato la **user flag**.

---
# Lateral movement 2 - da "jennifer" a "sa" | Ottenimento del file /etc/shadow → unshadow attack → sa

A questo punto ho iniziato veramente ad enumerare la macchina in modo approfondito.
Molto spesso procedo caricando strumenti come **linpeas** o **pspy64**, che velocizzano la raccolta dei dati potenzialmente critici o il monitoraggio di processi in esecuzione.

In questo caso, nella sessione di **jennifer**, ho caricato **pspy64**, che ha rivelato **uno script di backup** che veniva eseguito periodicamente da **root** tramite cronjob.

![pspy64.png](/images/imgs_sync/pspy64.png)

Facendo un _cat_ dello script ho potuto studiarne il funzionamento:

```bash
cat /usr/local/bin/backup.sh
```

![backup.sh.png](/images/imgs_sync/backup.sh.png)

`backup.sh` genera un **archivio zip** nella directory `/tmp`, contenente file sensibili tra cui `/etc/shadow`.

Da qui è stato sufficiente recuperare l'archivio in `/tmp` ed estrarlo:

```bash
unzip 1750674841.zip
```

Per ottenere il contenuto di **passwd** e **shadow** ed eseguire un **unshadow attack**:

```bash
unshadow passwd shadow > unshadow
john --format=crypt unshadow --wordlist=/usr/share/wordlists/rockyou.txt
```

**Switch utente**:

```bash
su sa
```

![lateral2-proof.png](/images/imgs_sync/lateral2-proof.png)

---
# Privilege Escalation | Cronjob abuse → root

A questo punto ho fatto un checkup dei passi fatti fino ad ora, notando che lo stesso script visto in precedenza (`backup.sh`) era modificabile dall'utente **sa**.

_Sapendo che lo script viene eseguito da root tramite cronjob, è sufficiente iniettare un comando malevolo nello script stesso per ottenere una chiara e semplice privilege escalation verticale, in questo caso impostando il **SUID bit** a **`bash`**:_

```bash
echo "chmod +s /bin/bash" >> /usr/local/bin/backup.sh
```

Dopo circa un minuto, lo script viene eseguito e chiamando una shell privilegiata otteniamo accesso a root:

```bash
/bin/bash -p
```

![system-proof.png](/images/imgs_sync/system-proof.png)

---
# Conclusioni finali

Come al solito, trovo le macchine di Vulnlab dei capolavori.
Fanno sempre scoprire tecniche nuove, strumenti alternativi e informazioni su scenari potenzialmente ritrovabili nel mondo reale.

In particolare, mi ha colpito la progressione molto lineare ma realistica dell’exploit chain:
dalla **data exfiltration** tramite **rsync**, **riutilizzo di password** al **password cracking** fino alla **privilege escalation abusando del cronjob** — tutte tecniche facilmente ritrovabili in scenari reali.

Ho impiegato parecchio tempo per rootare _sync_, e i miei migliori alleati per questo box sono stati:
-  **pspy64**, per individuare il cronjob
-  **linpeas**, per scoprire la possibilità di modificare lo script con l'utente **sa** (_dettaglio che mi ero completamente perso nonostante avessi già interagito con lo script_)

**Fonti**:

-  **Comandi utili per l'enumerazione del protocollo _rsync_ |** https://hackviser.com/tactics/pentesting/services/rsync
-  **Unshadow Attack (Youtube, Esadecimale) – un canale che mi ha insegnato tantissimo nella mia formazione in cybersecurity |** https://www.youtube.com/watch?v=eVlVQHlJC6U&list=PLJnLaWkc9xRiI6Uxygcxrsqlza3KhRy4v&index=8