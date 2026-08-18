+++
date = '2026-08-16T11:06:34+02:00'
draft = true
title = 'Baby Frame Writeup EN'
+++
**Author**: **`joy.scd01`**

**Date**: **`15/08/2026`**

![pwn.png](/images/imgs_babyframe/pwn.png)

---
# Introduction

**_Baby Frame_** is a fascinating challenge in the **Satellite Exploitation** track path on **Hack The Box**. Unlike traditional web or system exploitation, this track requires a deep dive into real-world space communication protocols.

The objective of this challenge is to successfully communicate with a simulated satellite to execute a **`HEALTHCHECK`** command and retrieve the flag. To achieve this, we must construct valid space packets from scratch by studying the **CCSDS (Consultative Committee for Space Data Systems)** standards and implementing the correct binary structures in the provided **`client.py`** script.

---
# Background: CCSDS Space Protocols

Before diving into the code, it's crucial to understand how satellites receive commands from a Ground Station.
Space communications rely on layers of encapsulation. For this challenge, we need to understand two main structures:
- [_CCSDS Space Packet Protocol_](https://ccsds.org/Pubs/133x0b2e2.pdf)
- [_CCSDS TC Space Data Link Protocol_](https://ccsds.org/Pubs/232x0b4e1c1.pdf)

## 1. Space Packet Protocol (SPP)
The **Space Packet** is the core data unit. It encapsulates the actual application data (in our case, the **`HEALTHCHECK`** string). It consists of a **Packet Primary Header** (6 bytes), which includes:

**`a)`** **1-2 Bytes**:
- **`Packet Version Number | (3 bit)`: It indicates the protocol version (usually 0)** 
- **`Packet Type | (1 bit)`: 0 = Telemetry(TM), 1 = Telecommand(TC)**
- **`Secondary Header Flag | (1 bit)`: Indicates whether a secondary header is present**
- **`Application Process ID (APID) | (11 bit)`: Identifies the destination application**

**`b)`** **3-4 Bytes**
- **`Sequence Flags | (2 bit)`: 0 = continuation, 1 = first segment, 2 = last segment, 3 = unsegmented**
- **`Packet Sequence Count | (14 bit)`: Packet counter**

**`c)`** **5-6 Bytes**
- **`Packet Data Length (16 bit)`: Length of the packet data**

![spp.png](/images/imgs_babyframe/spp.png)

## 2. Telecommand (TC) Transfer Frame
Satellites don't just accept raw **Space Packets**; they need to be transmitted over a data link. The Space Packet must be encapsulated inside a **TC Transfer Frame**. 
This frame includes a **Transfer Frame Primary Header** (5 bytes), which includes:

**`a)` 1-2 Bytes**:
- **`Transfer Frame Version Number | (2 bit)`: Usually 0**
- **`Bypass Flag | (1 bit)`: 0 = use control, 1 = bypass**
- **`Control Command Flag | (1 bit)`: 0 = data, 1 = control**
- **`Reserved Spare | (2 bit)`: Must be 0**
- **`Spacecraft ID | (10 bit)`: Identifies the satellite**

**`b)` 3-4 Bytes**:
- **`Virtual Channel ID | (6 bit)`: Identifies the virtual channel**
- **`Frame Length | (10 bit)`: Frame length**

**`c)` 5 Byte**:
- **`Frame Sequence Number | (8 bit)`: Counter that identifies the order of frames within a virtual channel**

![ttf.png](/images/imgs_babyframe/ttf.png)

Our task is to craft both the **Space Packet** and the **TC Transfer Frame** sequentially to deliver our payload.

---
# Code Analysis & Exploitation

The challenge provides us with a python script (**`client.py`**) acting as our **Ground Station**. However, the functions responsible for generating the packets are empty.

## Step 1: Crafting the Space Packet
First, we need to build the **Space Packet** containing our **`HEALTHCHECK`** payload. According to the **CCSDS** standards, we format the header by shifting bits to their correct positions to match the required field lengths using bitwise operations in Python.

**Note**: _Since this challenge is still **active**, I won't provide the exact copy-paste script. Instead, here is the theoretical breakdown of how to construct the **6-byte Primary Header**:_

*   **Bytes 1 & 2 (Packet ID & Flags):**
    We need to pack the **Version** (**`0`**), **Type** (**`1 for TC`**), **Secondary Header Flag** (**`0`**), and the **`11-bit`** **APID**. 
    To form the first byte, we bit-shift the **Version** left by 5 (**`<< 5`**), the **Type** by 4 (**`<< 4`**), and the **Header Flag** by 3 (**`<< 3`**). Since the **APID** is 11 bits long, its top 3 bits must fit into this first byte, so we shift it right by 8 (**`>> 8`**) and use the bitwise `OR` (`|`) to combine everything.
    The second byte will simply contain the remaining 8 bits of the **APID**, which we extract using a bitwise `AND` mask (`& 0xFF`).

*   **Bytes 3 & 4 (Sequence Control):**
    These bytes contain the **Sequence Flags** (To properly understand this field, I suggest you to read page **34-35** from the **PDF**) and the 14-bit **Packet Count**.
    Similar to the first step, the third byte is built by shifting the **Sequence Flag** left by 6 (**`<< 6`**) and combining it with the upper 6 bits of the **Packet Count**.
    The fourth byte takes the remaining lower 8 bits of the **Sequence Count** (**`& 0xFF`**).

*   **Bytes 5 & 6 (Packet Length):**
    According to the **CCSDS** standard, the length field must be the total number of octets in the **Packet Data Field minus 1**.
    We calculate **`len(payload) - 1`**. The fifth byte takes the upper 8 bits (**`>> 8 & 0xFF`**), and the sixth byte takes the lower 8 bits (**`& 0xFF`**).

Finally, we concatenate these 6 generated bytes with our raw payload to form the complete **Space Packet**.

## Step 2: Crafting the TC Transfer Frame
Once the **Space Packet** is ready, we must encapsulate it into a **TC Transfer Frame** to be transmitted over the data link. The **Transfer Frame Primary Header** is 5 bytes long. Here is how we construct it using bitwise operations:

*   **Bytes 1 & 2 (Flags & Spacecraft ID):**
    The first byte contains several flags: **Transfer Frame Version Number** (**`0`**), **Bypass Flag** (**`0`**), **Control Command Flag** (**`0`**), and **Reserved Spare** (**`0`**). We shift these to their respective positions (**`<< 6`**, **`<< 5`**, **`<< 4`**, **`<< 2`**). The **Spacecraft ID (SCID)** is 10 bits long, so we extract its top 2 bits (**`>> 8 & 0x03`**) and use a bitwise **`OR`** to pack them into the end of this first byte.
    The second byte simply contains the remaining 8 bits of the **SCID** (**`& 0xFF`**).

*   **Bytes 3 & 4 (Virtual Channel ID & Frame Length):**
    The third byte starts with the 6-bit **Virtual Channel ID (VCID)**, shifted left by 2 (**`<< 2`**). 
    Next is the **Frame Length** field (10 bits). According to the **CCSDS** standard, the length must be the **total length of the frame** (5 bytes of header + length of our **Space Packet** payload) **minus 1**. We take the top 2 bits of this calculated length to complete the third byte (**`>> 8 & 0x03`**), and place the remaining 8 lower bits into the fourth byte (**`& 0xFF`**).

*   **Byte 5 (Frame Sequence Number):**
    Unlike the **Space Packet** which uses a complex sequence control field, the fifth byte of the **TC Frame Header** is simply an 8-bit sequence counter. We just mask our counter variable with `& 0xFF` to ensure it fits perfectly into a single byte.

Finally, we concatenate these 5 header bytes with our payload (which is the fully constructed **Space Packet** from Step 1) to form the complete **TC Transfer Frame**.

## Step 3: Execution

With the protocol implementation complete, we execute the script to connect to the **HTB** instance and send our fully encapsulated telecommand.

```bash
python3 client.py
```

The satellite successfully decodes the **Transfer Frame**, extracts the **Space Packet**, processes the **`HEALTHCHECK`** command, and returns the flag.

![flag.png](/images/imgs_babyframe/flag.png)

---
# Final Thoughts

**_Baby Frame_** is an excellent entry point into the world of satellite hacking. 

It strips away the traditional layers of operating system security and forces you to interact directly with raw network engineering and space data link protocols. Understanding how to read the **CCSDS Blue Books** and translate those binary structures into **Python** code is a fundamental skill in the **Satellite Exploitation** branch and the rest of the **Satellite** track path.

**Sources**:

- **CCSDS Space Packet Protocol | https://ccsds.org/Pubs/133x0b2e2.pdf**

- **CCSDS TC Space Data Link Protocol | https://ccsds.org/Pubs/232x0b4e1c1.pdf**