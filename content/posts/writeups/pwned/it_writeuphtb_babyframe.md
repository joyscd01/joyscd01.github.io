+++
date = '2026-08-16T11:06:38+02:00'
draft = true
title = 'Baby Frame Writeup IT'
+++
**Author**: **`joy.scd01`**

**Date**: **`15/08/2026`**

![pwn.png](/images/imgs_babyframe/pwn.png)

---
# Introduzione

**_Baby Frame_** è un'affascinante challenge del track path **Satellite Exploitation** su **Hack The Box**. A differenza delle tradizionali vulnerabilità web o di sistema, questo percorso richiede un'analisi approfondita dei protocolli di comunicazione spaziale impiegati nel mondo reale.

L'obiettivo di questa challenge è comunicare con successo con un satellite per eseguire il comando **`HEALTHCHECK`** e recuperare la flag. Per farlo, dobbiamo costruire da zero dei pacchetti spaziali, studiando gli standard del **CCSDS (Consultative Committee for Space Data Systems)** e implementando le corrette strutture binarie nello script **`client.py`** fornito.

---
# Background: Protocolli Spaziali CCSDS

Prima di tutto, è fondamentale capire in che modo i satelliti ricevono i comandi da una **Ground Station**.
Per questa challenge, dobbiamo comprendere due strutture principali:
- [_CCSDS Space Packet Protocol_](https://ccsds.org/Pubs/133x0b2e2.pdf)
- [_CCSDS TC Space Data Link Protocol_](https://ccsds.org/Pubs/232x0b4e1c1.pdf)

## 1. Space Packet Protocol (SPP)
Lo **Space Packet** incapsula i dati effettivi dell'applicazione (nel nostro caso, la stringa **`HEALTHCHECK`**). È composto da un **Packet Primary Header** (6 byte), che include:

**`a)`** **1-2 Byte**:
- **`Packet Version Number | (3 bit)`: Indica la versione del protocollo (solitamente 0)** 
- **`Packet Type | (1 bit)`: 0 = Telemetry(TM), 1 = Telecommand(TC)**
- **`Secondary Header Flag | (1 bit)`: Indica la presenza di un header secondario**
- **`Application Process ID (APID) | (11 bit)`: Identifica l'applicazione di destinazione**

**`b)`** **3-4 Byte**
- **`Sequence Flags | (2 bit)`: 0 = continuation, 1 = first segment, 2 = last segment, 3 = unsegmented**
- **`Packet Sequence Count | (14 bit)`: Contatore dei pacchetti**

**`c)`** **5-6 Byte**
- **`Packet Data Length (16 bit)`: Lunghezza dei dati del pacchetto**

![spp.png](/images/imgs_babyframe/spp.png)

## 2. Telecommand (TC) Transfer Frame
I satelliti non accettano semplicemente **Space Packet** grezzi; questi devono essere trasmessi su un data link. Lo **Space Packet** deve quindi essere incapsulato all'interno di un **TC Transfer Frame**. 
Questo frame include un **Transfer Frame Primary Header** (5 byte), composto da:

**`a)` 1-2 Byte**:
- **`Transfer Frame Version Number | (2 bit)`: Solitamente 0**
- **`Bypass Flag | (1 bit)`: 0 = use control, 1 = bypass**
- **`Control Command Flag | (1 bit)`: 0 = data, 1 = control**
- **`Reserved Spare | (2 bit)`: Deve essere 0**
- **`Spacecraft ID | (10 bit)`: Identifica il satellite**

**`b)` 3-4 Byte**:
- **`Virtual Channel ID | (6 bit)`: Identifica il canale virtuale**
- **`Frame Length | (10 bit)`: Lunghezza del frame**

**`c)` 5 Byte**:
- **`Frame Sequence Number | (8 bit)`: Contatore che identifica l'ordine dei frame all'interno di un canale virtuale**

![ttf.png](/images/imgs_babyframe/ttf.png)

Il nostro compito è quello di generare sia lo **Space Packet** che il **TC Transfer Frame** per consegnare il payload.

---
# Code Analysis & Exploitation

La challenge ci fornisce uno script **Python** (**`client.py`**) che funge da **Ground Station**. Tuttavia, le funzioni incaricate di generare i pacchetti sono vuote.

## Step 1: Creazione dello Space Packet
Per prima cosa, dobbiamo costruire lo **Space Packet** contenente il nostro payload **`HEALTHCHECK`**. Secondo gli standard **CCSDS**, formattiamo l'header shiftando i bit nelle loro posizioni corrette per rispettare le lunghezze dei campi richieste, utilizzando le operazioni bitwise in **Python**.

**Nota**: _Essendo questa challenge ancora **attiva**, non pubblicherò lo script esatto copia-incolla. Di seguito propongo invece la spiegazione teorica di come costruire il **Primary Header da 6 byte**:_

*   **Byte 1 & 2 (Packet ID & Flags):**
    Dobbiamo impacchettare la **Version** (**`0`**), il **Type** (**`1 per TC`**), il **Secondary Header Flag** (**`0`**) e l'**APID** a **`11-bit`**. 
    Per formare il primo byte, shiftiamo a sinistra la **Version** di 5 (**`<< 5`**), il **Type** di 4 (**`<< 4`**) e l'**Header Flag** di 3 (**`<< 3`**). Poiché l'**APID** è lungo 11 bit, i suoi 3 bit più significativi devono rientrare in questo primo byte, quindi lo shiftiamo a destra di 8 (**`>> 8`**) e usiamo l'operatore bitwise `OR` (`|`) per unire il tutto.
    Il secondo byte conterrà semplicemente i restanti 8 bit dell'**APID**, che calcoliamo applicando una maschera bitwise `AND` (`& 0xFF`).

*   **Byte 3 & 4 (Sequence Control):**
    Questi byte contengono i **Sequence Flags** (per comprendere a pieno questo campo, suggerisco di leggere le pagine **34-35** del **PDF**) e il **Packet Count** a 14 bit.
    In modo simile al primo passaggio, il terzo byte viene costruito shiftando a sinistra il **Sequence Flag** di 6 (**`<< 6`**) e combinandolo con i 6 bit superiori del **Packet Count**.
    Il quarto byte prenderà i restanti 8 bit inferiori del **Sequence Count** (**`& 0xFF`**).

*   **Byte 5 & 6 (Packet Length):**
    Secondo lo standard **CCSDS**, il campo length deve indicare il numero totale di ottetti nel **Packet Data Field meno 1**.
    Calcoliamo quindi **`len(payload) - 1`**. Il quinto byte contiene gli 8 bit superiori (**`>> 8 & 0xFF`**), e il sesto byte prenderà gli 8 bit inferiori (**`& 0xFF`**).

Infine, concateniamo questi 6 byte generati con il nostro payload raw per formare lo **Space Packet** completo.

## Step 2: Creazione del TC Transfer Frame
Una volta pronto lo **Space Packet**, dobbiamo incapsularlo in un **TC Transfer Frame** per trasmetterlo sul data link. Il **Transfer Frame Primary Header** è lungo 5 byte:

*   **Byte 1 & 2 (Flags & Spacecraft ID):**
    Il primo byte contiene diversi flag: **Transfer Frame Version Number** (**`0`**), **Bypass Flag** (**`0`**), **Control Command Flag** (**`0`**) e **Reserved Spare** (**`0`**). Shiftiamo questi valori nelle rispettive posizioni (**`<< 6`**, **`<< 5`**, **`<< 4`**, **`<< 2`**). Lo **Spacecraft ID (SCID)** è lungo 10 bit, per cui estraiamo i suoi 2 bit superiori (**`>> 8 & 0x03`**) e usiamo un bitwise **`OR`** per inserirli alla fine di questo primo byte.
    Il secondo byte conterrà semplicemente i restanti 8 bit dello **SCID** (**`& 0xFF`**).

*   **Byte 3 & 4 (Virtual Channel ID & Frame Length):**
    Il terzo byte inizia con il **Virtual Channel ID (VCID)** a 6 bit, shiftato a sinistra di 2 (**`<< 2`**). 
    A seguire c'è il campo **Frame Length** (10 bit). Secondo lo standard **CCSDS**, la lunghezza deve corrispondere alla **lunghezza totale del frame** (5 byte di header + la lunghezza del nostro payload **Space Packet**) **meno 1**. Prendiamo i 2 bit superiori di questa lunghezza calcolata per completare il terzo byte (**`>> 8 & 0x03`**), e posizioniamo i restanti 8 bit inferiori nel quarto byte (**`& 0xFF`**).

*   **Byte 5 (Frame Sequence Number):**
    A differenza dello **Space Packet**, che usa un complesso campo di controllo della sequenza, il quinto byte del **TC Frame Header** è semplicemente un contatore di sequenza a 8 bit. Mascheriamo semplicemente la nostra variabile contatore con `& 0xFF` per assicurarci che si adatti perfettamente ad un singolo byte.

Per finire, concateniamo questi 5 byte dell'header con il nostro payload (che è lo **Space Packet** completato nello Step 1) per formare il **TC Transfer Frame** definitivo.

## Step 3: Esecuzione

Con l'implementazione del protocollo completata, eseguiamo lo script per connetterci all'istanza **HTB** e inviare il nostro telecomando completamente incapsulato.

```bash
python3 client.py
```

Il satellite decodifica con successo il **Transfer Frame**, estrae lo **Space Packet**, elabora il comando **`HEALTHCHECK`** e ci restituisce la flag.

![flag.png](/images/imgs_babyframe/flag.png)

---
# Considerazioni Finali

**_Baby Frame_** è un eccellente punto di ingresso nel mondo del satellite hacking.

Elimina i tradizionali livelli di sicurezza legati ai sistemi operativi e obbliga a interfacciarsi direttamente con concetti di ingegneria di rete pura e protocolli space data link. Comprendere come leggere i **CCSDS Blue Books** e tradurre quelle strutture binarie in codice **Python** è una competenza fondamentale nel ramo della **Satellite Exploitation** e per proseguire nell'omonimo track path.

**Fonti**:

- **CCSDS Space Packet Protocol | https://ccsds.org/Pubs/133x0b2e2.pdf**

- **CCSDS TC Space Data Link Protocol | https://ccsds.org/Pubs/232x0b4e1c1.pdf**
