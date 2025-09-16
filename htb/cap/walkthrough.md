HTB - [Machine Name] Walkthrough
Initial Enumeration
We start with a thorough Nmap scan to identify open ports and services on the target machine.

Bash

nmap -sV -p- 10.10.10.245
The scan reveals three open ports:

Port 21: FTP running vsftpd 3.0.3

Port 22: SSH running OpenSSH 8.2p1

Port 80: HTTP running Gunicorn, with a title of "Security Dashboard".

<br>

Foothold: Exploring the Web Server
The web server on port 80 shows a "Security Dashboard" and a /data directory. After navigating to /data, we find a .pcap file (packet capture) which we can download and analyze.

Bash

wget http://10.10.10.245/data/security_snapshot.pcap
We use a tool like Wireshark to inspect the .pcap file for any interesting information.

<br>

Gaining Access via FTP
Upon analyzing the packet capture, we discover credentials for the FTP service.

Username: nathan

Password: Buck3tH4TF0RM3!

We test these credentials and find that they also work for the SSH service on port 22, giving us a shell on the machine as the user nathan.

Bash

ssh nathan@10.10.10.245
With user access, we can now retrieve the user flag.

6e89061b7afba09dfa066e55e390c53d

<br>

Privilege Escalation
After gaining a user shell, our next step is to escalate privileges to root. We run LinPEAS to scan the system for potential vulnerabilities and misconfigurations.

Bash

# We can transfer linpeas.sh to the target machine using scp
scp linpeas.sh nathan@10.10.10.245:/tmp/
# Then execute it on the target
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
<br>

The LinPEAS output highlights that python3.8 has the cap_setuid capability, which allows it to change the User ID of a process. This is a significant finding that we can exploit to become root. We can use this capability to change our process's User ID to 0 (root) and then execute a root shell.

We write and execute a simple Python script to perform this action.

Python

import os
os.setuid(0)
os.system("/bin/bash")
After running the script, we successfully gain a root shell.

We can now read the contents of the root.txt file and claim the root flag.

69d7ea1977e866c3e9ba8c5ff6abe8f0
