---
title: "BombThreat @ Webverse"
date: 2026-06-24
categories: [Webverse, Medium]
tags: [sqli, mfa-bypass, jwt]
image:
  path: /assets/bomb.png
---

**Platform:** https://dashboard.webverselabs-pro.com/labs/bombthreat


BombThreat is a medium-difficulty lab running a Node.js + Express application backed by an in-memory SQLite database, fronted by an nginx gateway. The storyline drops you into a fictional "NEXUS Control" detonation portal. Your mission is to defuse a device before the countdown hits zero.

## Setup

```bash
➜ echo "10.100.0.62 bombthreat.local" | sudo tee -a /etc/hosts
```

## Recon

Navigating to `http://bombthreat.local` presents the NEXUS Control operator
sign-in page. No credentials, no registration link — just a login form.

<img width="1599" height="740" alt="image" src="https://github.com/user-attachments/assets/17c45c9b-e701-44fe-818d-6f6864eb8b4e" />


## SQL Injection — Bypassing Login

With no credentials in hand, I went straight for injection. The login endpoint
accepts JSON, so I sent the classic boolean bypass in the username field and
left the password empty.

```json
{"username":"' OR 1=1-- -","password":""}
```

<img width="1193" height="780" alt="image" src="https://github.com/user-attachments/assets/849f24f9-c3b1-4a70-8eef-8e8b08137dc2" />


That got me past the password gate and onto the Two-Factor Verification page.

## OTP Leak — MFA Response Debug Field

Before touching the OTP form, I looked back at the login response in Burp. The
server had accidentally shipped a `_debug` block in the JSON body — the OTP
sitting in plaintext, tagged with a TODO comment that never made it out before
production.

```json
{
  "success": true,
  "message": "Credentials accepted. MFA required.",
  "user": {
    "id": "usr_bg_0042",
    "email": "b.***goldstein@nexus-ctrl.io",
    "last_login": "2024-01-14T09:23:11Z",
    "clearance": "LEVEL 1"
  },
  "mfa": {
    "method": "EMAIL_OTP",
    "destination": "b.***goldstein@nexus-ctrl.io",
    "expires_in": 300,
    "_debug": {
      "note": "TODO: remove before production",
      "otp_value": "573627"
    }
  }
}
```

<img width="1419" height="633" alt="image" src="https://github.com/user-attachments/assets/6944a217-ce74-4915-8cfd-41c1dfc00018" />


I entered the OTP into the verification form and the server issued a JWT
access token, dropping me into the debug dashboard.

<img width="1599" height="822" alt="image" src="https://github.com/user-attachments/assets/c2bfadca-1365-457b-93c9-8dc3605cddb0" />


## JWT Forgery

The dashboard made it clear the deactivation endpoint requires **LEVEL 5**
clearance. Hitting `POST /api/bomb/deactivate` with my current token came back
with a 403.

```json
{
  "success": false,
  "error": "INSUFFICIENT_CLEARANCE",
  "message": "Deactivation requires LEVEL 5 clearance. You only have: level1 clearance."
}
```

<img width="1253" height="404" alt="image" src="https://github.com/user-attachments/assets/efabcc27-1213-44d1-ac74-e5ba1268b7cc" />


I decoded the jwt token on jwt.io, the clearance level is baked directly into the
payload.

<img width="1392" height="452" alt="image" src="https://github.com/user-attachments/assets/72703466-a885-42c7-b8d7-03ea3a7c98eb" />


The signing algorithm is specified in the JWT header, and if the server's parser blindly trusts whatever algorithm the token advertises, setting `"alg": "none"` removes the signature requirement entirely, the server accepts the token with no cryptographic check. I modified the header and changed the clearance claim to `level5`, generating an unsigned token.

<img width="1358" height="499" alt="image" src="https://github.com/user-attachments/assets/b3aacb4b-b4f1-47ea-9224-a824cafb4761" />

Replaying the deactivation request with the forged token:

<img width="1252" height="399" alt="image" src="https://github.com/user-attachments/assets/9a05402b-ebb6-4059-9297-30f26ee06539" />


```json
{
  "success": true,
  "message": "LEVEL 5 clearance accepted. Device deactivated.",
  "flag": "WEBVERSE{10M_xxxxxxxxx}"
}
```
