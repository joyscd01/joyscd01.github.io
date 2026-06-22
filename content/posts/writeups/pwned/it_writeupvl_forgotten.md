+++
date = '2026-05-23T18:26:33+02:00'
draft = false
title = 'Forgotten Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`05/06/2025`**

![forgotten_slide.png](/images/imgs_forgotten/forgotten_slide.png)

---
# Introduzione

**Forgotten** è una macchina Linux di livello _Easy_, interessante per la configurazione iniziale che prevede il setup di un **container locale MySQL**, da cui si accede all’istanza esposta di **LimeSurvey**.
Da qui, sfruttando l’upload di un **plugin malevolo** ho ottenuto l’accesso iniziale nel box.

Successivamente, un **riutilizzo di password** mi ha permesso di muovermi lateralmente dal container all’host. Infine, sfruttando i permessi e le directory condivise tra i due ambienti, ho effettuato un **container breakout** ottenendo così l’accesso a **root**.

---
# Tecniche Utilizzate

-  **Upload di plugin malevolo per ottenere Remote Code Execution**
-  **Riutilizzo di password**
-  **Container Breakout tramite permessi SUID**

---
# Enumerazione

## nmap

Scansione mirata con script e servizi:

```bash
nmap -sC -sV forgotten
```

![nmap.png](/images/imgs_forgotten/nmap.png)

**Porte aperte**:
-  **22/tcp** - SSH
-  **80/tcp** - HTTP

## HTTP - Web enumeration

Dato che la pagina web restituiva un errore indicando che non avevo i permessi per accedervi:

![web-server.png](/images/imgs_forgotten/web-server.png)

Ho eseguito **gobuster** per scannerizzare eventuali directory o percorsi nascosti.

```bash
gobuster dir -u http://forgotten/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gobuster.png](/images/imgs_forgotten/gobuster.png)

Tra i risultati è emerso un installer di **LimeSurvey** e la sua **versione**.

![installer.png](/images/imgs_forgotten/installer.png)

---

# Accesso Iniziale | LimeSurvey Setup → Upload Plugin Malevolo → RCE

**Nota**: _**LimeSurvey** è una piattaforma open-source per la gestione di sondaggi._

Seguendo le istruzioni sulla pagina di installazione, questa richiede di inserire i **dati di un database che si vuole utilizzare**.

Quindi, utilizzando un container docker, ho tirato su un **database locale MySQL**.
Per fare questo ho semplicemente cercato su Google: "**Come creare un database locale mysql con docker**".
L' **AI** del browser mi ha praticamente dato il comando spiegandomi anche le varie flag. 

Ho aggiunto solo i parametri richiesti nel form della pagina — _tutto molto intuitivo_.

```bash
sudo docker run --name limesurvey_db -e MYSQL_ROOT_PASSWORD=pentest -e MYSQL_DATABASE=limesurvey -e MYSQL_USER=limesurvey_user -e MYSQL_PASSWORD=pentest -p 3306:3306 -d mysql:latest
```

![db.png](/images/imgs_forgotten/db.png)

Ho inserito le informazioni richieste nel form e ho completato l'installazione.

![installer2.png](/images/imgs_forgotten/installer2.png)

Una volta fatto ciò, sono riuscito ad accedere al pannello amministrativo con le credenziali predefinite `admin` / `password`.

Dal pannello plugin, ho caricato un **plugin malevolo** che conteneva una **reverse shell PHP**. L’upload, l’attivazione e l’esecuzione del plugin mi hanno permesso di ottenere una shell come utente **`limesvc`**.

**Nota**: _Avendo già testato **LimeSurvey** in passato, ho riconosciuto la versione e sono andato a colpo sicuro con il plugin malevolo_.

In ogni caso, essendo a conoscenza della versione, basta cercare "**LimeSurvey 6.3.7 cve**" per trovare informazioni a riguardo.

Per questo exploit utilizzo sempre questa repo, seguendo le istruzioni:
- https://github.com/Y1LD1R1M-1337/Limesurvey-RCE

Quindi, dopo averla clonata, ho modificato il file **`php-rev.php`** inserendo l'_IP_ e la _Porta_:

![php-rev.png](/images/imgs_forgotten/php-rev.png)

Ho aggiunto la versione _6.3.7_ nel file **`config.xml`**:

![config.png](/images/imgs_forgotten/config.png)

Ho creato l'**archivio zip**:

![zip.png](/images/imgs_forgotten/zip.png)

Ho _caricato, installato e attivato_ il **plugin**:

![plugin.png](/images/imgs_forgotten/plugin.png)

A questo punto, dopo aver startato un **listener netcat**, ho navigato all'URL:

- http://forgotten/upload/plugins/Y1LD1R1M/

Per triggherare la **reverse shell**:

![initial-access.png](/images/imgs_forgotten/initial-access.png)

Qui ho notato già dal comando **`ifconfig`** un _Hostname_ e un _IP_ sospetti — classici segni di containerizzazione.

**Nota**: _Un indizio chiaro del fatto che ci si trovi in un container Docker è la presenza di interfacce come `eth0`, con indirizzi IP tipici della rete bridge predefinita. Inoltre, spesso manca un hostname completamente configurato._

Detto ciò, a questo punto ho stabilizzato la shell.
**Python** non era installato quindi per farlo ho provato altri comandi per la stabilizzazione.

Fonti: https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet

```bash
script -qc /bin/bash /dev/null
```

---
# Lateral Movement | Password Reuse → limesvc

Come al solito, ho iniziato ad enumerare la macchina con semplici comandi generali.
In particolare: **`env`**, per le variabili d'ambiente, ha rivelato la password dell'utente **limesvc**.

![env.png](/images/imgs_forgotten/env.png)

A questo punto mi sono connesso tramite SSH direttamente all'host principale.

![ssh-access.png](/images/imgs_forgotten/ssh-access.png)

All'interno della home ho trovato la **user flag**.

---
# Privilege Escalation | Container breakout → Permessi SUID → root

Qui ho avuto un momento di stallo totale.
Ero stanco e iniziavo a vedere doppio... 

Momento perfetto per approcciare un'**enumerazione incrociata tra container e host**.
.
.
.
![maxresdefault.jpg](/images/imgs_forgotten/maxresdefault.jpg)

(_In realtà c'è molto peggio_).

**Ho scoperto che**:

-  Lanciando `sudo -l` (**dentro il container**), potevo eseguire con l'utente **limesvc** qualsiasi comando come **root**, quindi ho switchato a **root** nel container.

![sudo-l.png](/images/imgs_forgotten/sudo-l.png)

-  Listando i file in **`/opt`** (**dall'host**), era presente una cartella scrivibile: **`limesurvey`**

Quindi, (**dall'host**) ho copiato il binario **`/usr/bin/bash`** in **`/opt/limesurvey`**.

(**dentro il container**) Sono andato nella path corrispondente:

```bash
cd /var/www/html/survey
```

E da qui ho modificato i permessi del binario **`bash`** per renderlo eseguibile come root:

```bash
chown root:root bash
chmod u+s bash
```

Infine, ho eseguito il binario (**dall'host**) forzando il mantenimento dei privilegi dell'utente owner, in questo caso **root**.

```bash
./bash -p
```

![root-proof.png](/images/imgs_forgotten/root-proof.png)

**Root Flag**.

Per mantenere l’accesso al sistema compromesso, ho inserito la mia **chiave pubblica SSH** all’interno del file `/root/.ssh/authorized_keys`.

---
# Conclusioni Finali

Come al solito, trovo le macchine di Vulnlab dei capolavori.
Fanno sempre scoprire tecniche nuove, tool alternativi e informazioni su scenari potenzialmente ritrovabili nel mondo reale.

Consiglio questo box. La fase di privilege escalation è forse un po' avanzata per una _Easy_, ma comunque soddisfacente. Mi ha permesso di approfondire e consolidare questa tecnica di **Container Breakout**, capendone meglio i limiti e i rischi associati a un ambiente containerizzato mal configurato.
Tra l'altro ho letto in altre guide che qualcuno ha utilizzato strumenti che vanno ad analizzare il container docker, automatizzando l'enumerazione che secondo il mio parere era la parte un po' più difficile, ma allo stesso tempo anche la più formativa — specialmente in un contesto di preparazione per **OSCP**, nel quale è fondamentale sviluppare il giusto modo di ragionamento, più che affidarsi a scorciatoie.

Oltre a questo è stato interessante vedere come poter configurare rapidamente un'**istanza MySQL locale** utilizzando **Docker**.

**Fonti**:

-  **Github repo per la creazione del plugin malevolo per LimeSurvey** | https://github.com/Y1LD1R1M-1337/Limesurvey-RCE
-  **Cheatsheet che ho trovato online contenente comandi per stabilizzare una shell** | https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet