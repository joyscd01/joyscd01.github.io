+++
date = '2026-05-20T17:53:59+02:00'
draft = false
title = 'Cap Writeup IT'
+++
**Nome:** `joy.scd01`

**Data:** `09/01/2025`

![Cap.png](/images/imgs_cap/Cap.png)

---
# Introduzione

**_Cap_** è una macchina **Linux** di difficoltà **Easy**.

L’analisi di un semplice file **PCAP** permette di recuperare credenziali sensibili e ottenere rapidamente un accesso iniziale al sistema.

La fase di Privilege Escalation invece si può ottenere abusando di una capability mal configurata che consente l’esecuzione di codice come **root**.

---
# Tecniche Utilizzate

- **IDOR (Insecure Direct Object Reference)**
- **Analisi PCAP** -> **Esfiltrazione Dati Sensibili**
- **Abuso di una Linux Capability mal configurata**

---
# Enumerazione

## Nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- cap -T4
```

![nmap1.png](/images/imgs_cap/nmap1.png)

Scansione mirata con script e rilevamento servizi:

```bash
nmap -sC -sV -T4 cap
```

![nmap2.png](/images/imgs_cap/nmap2.png)

**Porte aperte**:
- **21/tcp** → ftp
- **22/tcp** → ssh
- **80/tcp** → http
---
# HTTP - Enumerazione Web | http://cap:80

Visitando la web page ho notato la sezione **“Security Snapshot (5 Second PCAP + Analysis)”**, che mostra informazioni relative ai pacchetti di rete catturati.

![3.png](/images/imgs_cap/3.png)

Ci sono **due aspetti interessanti**:
- Cliccando sul pulsante **Download** è possibile scaricare un file contenente i pacchetti di rete (**`.pcap`**).
- Il contenuto cambia modificando l'identificativo numerico (ID) presente nell’URL:

_Questa è una chiara vulnerabilità **IDOR (Insecure Direct Object Reference)**: l'applicazione non verifica i controlli di autorizzazione, permettendo a chiunque di accedere a catture di rete precedenti semplicemente iterando l'ID._

![changes0.png](/images/imgs_cap/changes0.png)

Ho quindi scaricato il file **“0”** per analizzarlo con **Wireshark**.

---
# Accesso Iniziale | Analisi PCAP -> Esfiltrazione Dati Sensibili

Ho individuato traffico contenente **credenziali in chiaro**:

![creds.png](/images/imgs_cap/creds.png)

**Credenziali trovate**:
`nathan:Buck3tH4TF0RM3!`

Ho utilizzato quest'ultime per ottenere l’**accesso iniziale** alla macchina via **SSH**:

![ssh_nat.png](/images/imgs_cap/ssh_nat.png)

La **user flag** si trova in `/home/nathan`.

---
## Privilege Escalation | Abuso di una Linux Capability mal configurata

Per individuare una possibile escalation ho iniziato ad enumerare la macchina localmente.

```bash
getcap -r / 2>/dev/null
```

![cap_vuln.png](/images/imgs_cap/cap_vuln.png)

Ho notato che il binario **python3.8** ha una capability pericolosa assegnata: **``cap_setuid``**

A questo punto, come sono solito fare in questi casi, ho controllato su **gtfobins**:
- https://gtfobins.github.io/gtfobins/python/

Abusando di questa capability è possibile ottenere una **shell da root**:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash");'
```

![root.png](/images/imgs_cap/root.png)

La **root flag** si trova in `/root`.

---
# Conclusioni Finali

Uno dei primi box Linux che abbia affrontato: molto semplice, ma nel complesso ottimo per consolidare le basi di **enumerazione**, **analisi del traffico di rete** e **post-exploitation** in ambienti Linux.
È ideale per comprendere quanto anche informazioni apparentemente innocue possano diventare critiche se esposte in modo improprio, e quanto una singola misconfigurazione possa avere un impatto significativo sulla sicurezza di un sistema.

**Fonti**:

- GTFOBins | https://gtfobins.github.io/gtfobins/python/