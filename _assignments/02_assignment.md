---
type: assignment
date: 2026-09-19T8:00:00-04:00
enable: yes
title: 'Lab #2 - SmartLock V1'
due_event: 
    type: due
    enable: yes
    date: 2026-10-16T23:59:00-04:00
    description: 'Lab #2 due'
---

# The controller

You can find the starter code for this part [here](https://github.com/purs3lab/ECE59500-130-iot/tree/part2). Submit an `answers.txt` file with answers to the descriptive questions that you will find throughout this part (Q1 to Q8).

Currently, the only way to unlock the system is to manually enter the PIN on the keypad. But what if you wanted to unlock it without needing a PIN? This is where the controller comes in. Instead of entering a PIN, you will use a pre-authorized device that can be used to unlock the system when you are within a certain Bluetooth range.

For part1, we kept the init phase and the normal phase separate to clarify the different functionalities you were expected to implement. For the remaining parts, we will combine the functionalities.

This part touches every layer you've built so far: the keypad/LCD device code from part1, plus two new pieces - Bluetooth Low Energy (BLE) on the device, and a phone-facing webapp that talks to it. Because of that spread, the instructions below are organized into sections:
- Setup, the admin panel, and the asyncio rework everything else depends on - covered right below.
- [Pairing](#pairing) - the BLE GATT service and the pairing protocol (registering a phone with the lock).
- [Unlock](#unlock) - the challenge-response protocol a paired phone uses to actually unlock the door.
- [Mobile webapp](#mobile-webapp) - the phone-side webapp: Web Bluetooth, Web Crypto, and the pairing/unlock UI.

### Setup

**No new physical components.** The circuit is exactly what you built for [part1](https://github.com/purs3lab/secure-edge-iot-project/tree/part1) - keypad + I2C LCD. Everything new this part is firmware and web code.

`part2_starter_code/` is *not* a self-contained project - it only shows what's new or changed for this part. Concretely:
- `common.py`, `main.py`, `init.py` in `part2_starter_code/` contain the *new* pieces for this part (BLE UUIDs, the admin panel, pairing/unlock logic) layered on top of the file structure you already have. Functions you already implemented and solved in part1 - `pin_exists`, `store_pin`/`load_pin`, `constant_time_compare`, `validate_pin`, `get_input_string` - are **not reprinted here**. Bring your own working versions forward. The one thing that *does* change about them: see "Using asyncio" below, they all need to become non-blocking.
- `ecdsa_verify.py` is **brand new and given to you complete** - a small vendored elliptic-curve signature *verifier* for the device side. MicroPython has no ECDSA support at all, so this is infrastructure you consume, not an exercise. See "A public/private key primer" below for what it actually does.
- `index.html`, `mobile.css`, `mobile.js`, `ble_protocol.js` are **brand new** - the phone-facing pairing/unlock page, and the only page in this part's webapp. Covered in [Mobile webapp](#mobile-webapp).

**New MicroPython dependency: `aioble`.** BLE support on the Pico is split into a low-level `bluetooth` module (built into the firmware) and `aioble`, an asyncio-friendly wrapper library that is *not* preloaded onto the board - you install it the same way you'd install any [MicroPython package](https://docs.micropython.org/en/latest/reference/packages.html). With the Pico connected over USB and **not** currently holding a MicroPico REPL session (disconnect it first via `Ctrl+Shift+P` → `MicroPico: Disconnect`, otherwise the serial port is busy), run from a terminal on your computer:
```sh
pip install mpremote
mpremote mip install aioble
```
This fetches the package using *your computer's* internet connection and copies it onto the board over USB, so the Pico itself doesn't need WiFi active during install. Reconnect via MicroPico afterwards. You can browse what you just installed [on GitHub](https://github.com/micropython/micropython-lib/tree/master/micropython/bluetooth/aioble) - `aioble.Service`/`aioble.Characteristic`/`aioble.advertise` etc. are what `common.py`/`main.py` build on.

**A phone with Bluetooth.** You'll need your own phone (or a laptop with Bluetooth) to act as the controller. The webapp that talks to it needs to run in a browser that implements the [Web Bluetooth API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API) - see [Mobile webapp](#mobile-webapp) for exactly which browsers that means.

If you've never worked with Bluetooth Low Energy before, it's worth reading a general primer before diving into the protocol-specific detail in the [Pairing](#pairing) section - [Adafruit's "Introduction to Bluetooth Low Energy"](https://learn.adafruit.com/introduction-to-bluetooth-low-energy) covers the GATT services/characteristics model this whole part is built on.

### What you'll build

- An **admin panel** on the keypad/LCD, gated behind your existing unlock PIN.
- A **BLE GATT service** on the device advertising a handful of characteristics (device side of both protocols below).
- The **pairing protocol**: from the admin panel, put the lock into pairing mode; a phone connects over BLE, proves it holds a private key, and gets registered. Full detail in [Pairing](#pairing).
- The **unlock protocol**: a paired phone, in range, unlocks the door via a fresh challenge-response exchange every time - no static "I'm paired" signal. Full detail in [Unlock](#unlock).
- The **mobile webapp**: the page for pairing new phones and unlocking - no server, no cloud, just the browser talking to the lock over BLE. Full detail in [Mobile webapp](#mobile-webapp).
- Migrating your existing lock code from blocking calls to **asyncio**, so BLE and the keypad can run concurrently (see below).

### Admin panel

First thing you need to implement is an admin panel that can be accessed by typing a non-numeric key on the keypad - specifically the `A` key (`ADMIN_PANEL_ENTRY_CODE` in `common.py`). Note that this panel should prompt for the current PIN before giving access, using the exact same `validate_pin()` you already have - one successful check authorizes the whole admin session. The other non-numeric keys are reused as menu navigation once you're inside the panel: `B`/`C` to page back/forward through multi-item menus (there are more admin options than what can fit on a 2-line LCD at once), `*` to cancel/back out.

The following functionalities are expected through the admin panel:
- **Reset PIN**: this is the `reset_pin()` flow you already built in part1 - nothing new to implement here, just relocate it so it only runs from inside the admin panel instead of automatically at boot.
- **Pair device**: pair a new controller. See [Pairing](#pairing).
- **Unpair device**: remove a paired controller. Also in [Pairing](#pairing).
- **Exit**: go back to normal lock operation.

The admin panel is gated behind the unlock PIN.

> **Q1.** Why does the admin panel need to be behind the PIN at all? And why does that matter *more* now, in this part, than a guessed PIN did back in part1? Part1 never added a lockout after repeated wrong PIN attempts at the door - given your answer, would you add an attempt-lockout or cooldown at the admin-panel entry point? This lab doesn't require one, but you should be able to justify your choice either way.

### Using asyncio

When you add bluetooth capability, you might need to update existing code to work alongside the bluetooth code. You don't want the normal lock functionality to wait on a bluetooth connection and vice versa. One way to do this will be to make the bluetooth part run on a different thread while the "normal" functionality runs on the main thread. But due to the way "normal" functionality interact with the bluetooth part, this might get complex or might introduce bugs that are hard to debug. The suggested idiomatic way is to run the bluetooth advertising/connection handling as a background asyncio task (`asyncio.create_task()`), while the normal lock functionality runs as the main coroutine under a single `asyncio.run()` call. Because both share the same event loop, whenever either one hits an `await` the other gets a chance to run - no threads needed.

This means converting code you already wrote: every blocking `time.sleep(n)` in your part1 code needs to become `await asyncio.sleep(n)`, and every function in the call chain down to the keypad loop needs to become `async def`. This isn't optional - a plain `time.sleep()` anywhere blocks the *entire* single-threaded event loop for its duration, which means BLE advertising and any in-progress GATT connection would freeze too, right along with the keypad. At minimum, expect to convert `get_input_string`, `reset_pin`/`set_pin`, and your keypad loop itself.

### A public/private key primer

Since we don't want unauthorized devices connecting to the lock, we will be using a public-private key signing mechanism that the mobile uses to prove its identity.

The short version: in [public-key (asymmetric) cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography), a keypair has two halves - a private key you never share, and a public key you can hand out freely. A [digital signature](https://en.wikipedia.org/wiki/Digital_signature) is something only the private key holder can produce, but that *anyone* holding the public key can verify. That gives us exactly the property we need: the phone can prove "I'm the same device that registered this public key" without ever transmitting anything secret over the air.

This project specifically uses **ECDSA over the P-256 curve, with SHA-256** - the same combination TLS/HTTPS itself commonly uses. A few reasons this particular combination was chosen, in case you're curious or evaluating alternatives for your own projects:
- MicroPython's `hashlib` on the Pico doesn't include SHA-512, which algorithms like Ed25519 need internally. Sticking to SHA-256 means the device only needs to vendor the elliptic-curve math itself, not a second new hash implementation.
- The RP2350 (the Pico 2's chip) has a hardware-accelerated SHA-256 engine, so this choice is also just faster.
- The browser's native [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto) supports ECDSA P-256 out of the box, and conveniently, `crypto.subtle.sign()` for ECDSA returns the signature as raw concatenated `r || s` bytes rather than [ASN.1/DER](https://en.wikipedia.org/wiki/X.690)-encoded - so there's no signature parsing/re-encoding needed on either side of the connection.

If you want real background on how elliptic-curve signatures work under the hood (not required to complete this lab, but good to know exists), Cloudflare's [primer on elliptic curve cryptography](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography/) is a good place to start, and the [Wikipedia ECDSA page](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) covers the algorithm itself.

### Pairing

This covers the admin-panel "Pair device" / "Unpair device" flow: registering a phone's public key with the lock over BLE, and later removing it. The other half - what a *paired* phone does to actually unlock the door - is in [Unlock](#unlock).

#### BLE/GATT background

If you haven't used Bluetooth Low Energy before, read [Adafruit's introduction](https://learn.adafruit.com/introduction-to-bluetooth-low-energy) first; this section only covers the specific pieces this project touches.

A BLE peripheral (the lock) exposes a **GATT** tree: one or more **services**, each containing one or more **characteristics**. A characteristic is roughly "a named slot that holds bytes" - a central (the phone) can `read` it, `write` it, or subscribe to be `notify`-ed when it changes, depending on how the peripheral declared it. Every service and characteristic is identified by a **UUID**. Standard, Bluetooth-SIG-defined characteristics (battery level, device name, etc.) use well-known UUIDs; anything custom - like this whole lock service - needs one you pick yourself.

You'll declare your service and characteristics in `common.py` using `aioble.Service(...)` and `aioble.Characteristic(...)`/`aioble.BufferedCharacteristic(...)`, then call `aioble.register_services(...)` **once**, before `aioble.advertise()` is ever called - characteristics can't be added to a service after it's registered.

This project needs one service and nine characteristics on it, covering pairing (`PAIR_CODE`, `PAIR_PUBKEY`, `PAIR_NAME`, `PAIR_SIG`, `PAIR_COMMIT`, `PAIR_STATUS`) and unlock (`CHALLENGE`, `UNLOCK_RESPONSE`, `UNLOCK_STATUS` - see [Unlock](#unlock)). Exact byte formats for each are documented in `ble_protocol.js`, which both sides have to agree with.

**Picking UUIDs**: each characteristic needs its own UUID, distinct from the service's and from each other, and not colliding with the standard Generic Access/Attribute characteristics already present in every GATT table (`0x1800`-`0x1801`, `0x2800`-`0x2803`, `0x2900`-`0x2905`, `0x2A00`, `0x2A01`, `0x2A05`). A real device's GATT attribute table has finite space, and nine characteristics as full 128-bit UUIDs can overflow it - on this hardware, that failure mode is *silent*, so if a characteristic mysteriously "isn't there" from the phone side, this is worth checking.

Whatever UUIDs you pick in `common.py` **must match exactly** in `ble_protocol.js` on the webapp side - that file's `CHAR` object is the single source of truth the phone builds its GATT client against.

**`write`/`capture`/`read`/`notify` flags**, when constructing each `aioble.Characteristic(...)`:
- `write=True` lets the phone write into it. Add `capture=True` so your device code can `await characteristic.written()` to be notified the instant a write lands, instead of having to poll.
- `read=True` lets the phone read its current value - but aioble has no "on read" hook. The device is entirely responsible for calling `characteristic.write(value)` itself ahead of time to set whatever the *next* read should return (this is how `CHALLENGE` gets a fresh nonce before each read - see [Unlock](#unlock)).
- `notify=True` lets the phone subscribe to be pushed an update instead of polling with reads - used for `PAIR_STATUS`/`UNLOCK_STATUS` specifically because of the race condition described below.

**The 20-byte truncation trap**: MicroPython's default max characteristic value size is 20 bytes. A write larger than that is **silently truncated** to 20 bytes rather than rejected or raising an error. `PAIR_PUBKEY` (65 bytes), `PAIR_SIG` (64 bytes), `PAIR_NAME`, and the unlock protocol's `UNLOCK_RESPONSE` (72 bytes) are all wider than that. If you declare these with plain `aioble.Characteristic(...)`, writes to them get cut off with no error on either side, and you'll only notice several steps downstream when signature verification fails for no apparent reason. Use `aioble.BufferedCharacteristic(..., max_len=N)` instead for any characteristic that needs to carry more than 20 bytes, with `max_len` set to the field's actual byte length (65 for `PAIR_PUBKEY`, 32 for `PAIR_NAME`, 64 for `PAIR_SIG`, 72 for `UNLOCK_RESPONSE`).

**The status-read race, and why `PAIR_STATUS`/`UNLOCK_STATUS` use `notify`**: a GATT "write with response" only confirms the *write* was received - not that the device has finished processing it (parsing, hashing, verifying a signature all take real time). If the phone immediately turns around and reads `PAIR_STATUS` right after its `PAIR_COMMIT` write, it can easily race the device and read a stale or empty value. The fix used throughout this project: the phone subscribes to notifications on the status characteristic *before* triggering the action, then waits for the notification the device fires once it's actually done, via `characteristic.write(data, send_update=True)`. Never read-immediately-after-write for these two characteristics - always subscribe-then-wait-for-notify. You'll need the same pattern on the phone side in `mobile.js` - see [Mobile webapp](#mobile-webapp).

#### The pairing protocol

Preconditions: the owner has already authenticated into the admin panel with the unlock PIN, and selected "Pair device."

1. **Device generates a one-time code.** Six random decimal digits shown on the LCD.
2. **Owner reads the code off the LCD**, opens the mobile webapp, taps "Pair New Lock," and picks the lock from the browser's Bluetooth device picker.
3. **Phone establishes (or reuses) its identity.** First time pairing anything, the phone generates its own ECDSA P-256 keypair via the Web Crypto API and stores it locally (never transmitted). This same keypair gets reused for every lock this phone pairs with afterward - see [Mobile webapp](#mobile-webapp).
4. **Phone signs a message and sends the pairing bundle.** The owner types the 6-digit code into the webapp. The phone builds the byte string `LOCKID + ":PAIR:" + code` (see "Domain separation" below for why the `":PAIR:"` tag is there) and signs it with its private key. It then writes, in order: the code to `PAIR_CODE`, its raw public key to `PAIR_PUBKEY`, a display name to `PAIR_NAME`, the signature to `PAIR_SIG`, and finally an empty value to `PAIR_COMMIT` - which is the signal that tells the device "I'm done, go validate everything I just sent."
5. **Device validates on commit.** On the `PAIR_COMMIT` write, the device: checks a pairing session is actually pending; compares the received code against the one it generated using `constant_time_compare`; if that matches, verifies the signature against the received public key using `ecdsa_verify.verify()`; if *that* matches, sanitizes the received name, computes a `device_id` as the first 8 bytes of `sha256(pubkey)` (hex-encoded), and appends `{device_id, pubkey, name}` to local storage, capped at `MAX_PAIRED_DEVICES`.
6. **Device reports the result** both back over BLE and via an LCD message. A successful `PAIR_STATUS` must include the new `device_id` - it's the only way the phone ever learns it, and every unlock attempt from here on needs to send it back (see [Unlock](#unlock)).

If the owner wants out before finishing, pressing `*` on the keypad should cancel the session outright.

#### Domain separation: the `":PAIR:"` / `":UNLOCK:"` tags

Every signed message in this project is built as `LOCKID + <tag> + <the rest of the fields>`, where `<tag>` is a fixed, purpose-specific literal - `":PAIR:"` for pairing, `":UNLOCK:"` for unlocking (see [Unlock](#unlock)). Both sides - `common.py`'s `TAG_PAIR`/`TAG_UNLOCK` and `ble_protocol.js`'s matching constants - have to agree on these tags byte-for-byte, since each side independently reconstructs the same message and checks it against the same signature; neither side transmits "the message" as such, only the fields it's built from.

> **Q2.** A signature only proves "the private key holder signed *exactly this byte string*" - it says nothing about what that byte string was *for*. Given that, what could go wrong if the pairing message and the unlock message were ever built from the same bytes (no tag, no `LOCKID`)? This general pattern is called **domain separation** - why does it generalize to *any* situation where one keypair signs more than one kind of message?

#### What the code proves vs. what the signature proves

Two separate checks happen during pairing: the device compares the typed 6-digit code, and independently verifies the phone's signature.

> **Q3.** These two checks protect against different things. What does the 6-digit code actually prove (and *not* prove)? What does the signature prove instead - what's this property called, and why does it matter here even though the *unlock* protocol (see [Unlock](#unlock)) re-verifies a signature from scratch on every single unlock attempt anyway?

#### Why not BLE's own pairing (Just Works / Passkey)?

Bluetooth's Security Manager Protocol - the "pairing" prompt you get connecting a keyboard or headphones - has its own menu of **association models**, negotiated from the I/O capabilities each side declares:
- **Just Works**: negotiates encryption but authenticates neither side - fine against passive eavesdropping, but does nothing against an active man-in-the-middle during the handshake itself. It's the fallback whenever either side reports it has no display/keyboard.
- **Passkey Entry**: one side displays a 6-digit passkey, the other side's user types it in - in shape, almost exactly what `PAIR_CODE` already does above. (aioble does expose this - `aioble.security.pair(..., io=...)` with a display-capable `io` value - if you go looking.)

Given the lock already has an LCD, it's reasonable to ask why this project doesn't just call `aioble.security.pair()` with Passkey Entry and let BLE bonding do this work, instead of the manual `PAIR_CODE`/`PAIR_SIG`/`PAIR_COMMIT`. Two reasons:

1. **It can't actually be gated behind the PIN.** The lock has to stay BLE-advertising and connectable continuously so the *unlock* protocol works at any time (see [Unlock](#unlock)) - it isn't only reachable during a pairing window. The only thing that restricts pairing to a PIN-authenticated window is the device's own `pairing session pending` flag, set from the admin panel and checked when `PAIR_COMMIT` lands. BLE's Security Manager has no hook into that: bonding happens at the stack level, so a phone could connect and run the Passkey Entry dance the moment the lock is powered, whether or not anyone ever typed the door PIN. To make a native passkey bond mean "the PIN was already checked," you'd have to reject/allow it from your own code anyway (e.g. in the `_IRQ_PASSKEY_ACTION` handler, checking that same session flag) - at which point you're doing the identical app-layer gating work, just wrapped around a mechanism you don't otherwise control.
2. **Web Bluetooth can't drive it anyway.** `aioble.security.pair()` is a device-side API with no browser-side equivalent - the Web Bluetooth spec gives a page no way to initiate pairing, choose an association model, or display/read a passkey. If bonding happens at all, it's handled invisibly by the OS, outside `mobile.js`'s control or even visibility. Doing the actual proof-of-identity at the GATT/application layer instead - the code, the ECDSA signature over `LOCKID + ":PAIR:" + code`, the explicit `PAIR_COMMIT` - is what lets `mobile.js` drive the whole flow itself, deterministically, on any browser that implements Web Bluetooth, instead of depending on however each OS happens to implement SMP.

The consequence: the BLE link itself is never authenticated, and not necessarily encrypted, by the Bluetooth stack in this project - at best it'd be Just Works if a platform bonds at all underneath the app. That's fine, because that job is pushed up a layer: everything that actually needs to be trustworthy - who's connecting, whether they hold the right private key, whether a signed message is fresh - is checked in the code you write, not left to the BLE stack.

#### A note on the display name

`PAIR_NAME` is text typed by whoever is pairing - not validated by construction - and it doesn't stay in one place: it flows into `lcd.message()` on-device, and later gets rendered in the mobile webapp's paired-devices list and activity log (see [Mobile webapp](#mobile-webapp)).

> **Q4.** `lcd.message()` treats `\n` as a line separator. What could a maliciously-chosen `PAIR_NAME` do to the LCD if it isn't sanitized before being displayed? Separately, what's the risk on the webapp side once this same string gets rendered there? Should you sanitize it once, at the point it's first received, or again at each place it's rendered - why?

#### Removing a paired device

An admin panel option that lets you remove already registered controllers. This only removes the registered controller from within the lock's disk. You have to implement something similar on the WebApp side to remove a controller there.

### Unlock

This covers what happens every time an *already-paired* phone wants to unlock the door.

Unlike pairing, this protocol runs continuously in the background for the lifetime of the program - it's not gated behind the admin panel or PIN entry, and isn't a one-time event. A paired phone should be able to unlock the door any time it's in range and the device is powered, independent of whatever the keypad is doing.

#### Fresh challenge every time

This uses [challenge-response authentication](https://en.wikipedia.org/wiki/Challenge%E2%80%93response_authentication): the device issues a fresh, unpredictable challenge, and the phone must sign *that specific challenge*, every single time.

> **Q5.** Why does the challenge need to be fresh and unpredictable on every single unlock, instead of the phone just resending the same signed "I'm paired" message it used last time?

#### The protocol

1. **Device issues a nonce.** Every time `CHALLENGE` is read, the device has already generated and written a fresh random 32-byte nonce (`get_salt(32)`) for that connection - remember from [Pairing](#pairing) that `read`-only characteristics have no "on read" hook in aioble, so this has to be proactive: rotate the nonce, *then* let it be read, on a loop for as long as a connection is open.
2. **Phone signs it.** The phone reads `CHALLENGE`, then builds and signs `LOCKID + ":UNLOCK:" + nonce + device_id`.
3. **Phone writes the response.** `device_id` (8 raw bytes) concatenated with the signature (raw `r‖s`, 64 bytes) = 72 bytes total, written to `UNLOCK_RESPONSE` in one shot.
4. **Device verifies and responds.** Look up the stored public key for the claimed `device_id`; verify the signature against **the exact nonce most recently issued on this connection**; on success, unlock. Report the result via `UNLOCK_STATUS`.

A nonce that nobody responds to (phone disconnected mid-exchange, signature never arrives, etc.) shouldn't sit around forever - give it a lifetime after which the device gives up waiting and rotates to a new one anyway.

#### Concurrency: this runs alongside the keypad, not instead of it

Both unlock paths - keypad PIN and BLE challenge-response - ultimately drive the same physical lock, which can only be in one state at a time. If a keypad unlock and a BLE unlock ever land close together, their "unlocked - wait a few seconds - re-lock" sequences must not run concurrently and race each other over what the LCD should say. Serialize *that specific sequence* with a single `asyncio.Lock()` shared between both unlock paths; every other LCD write is fine to interleave freely, since those are cosmetic, not a correctness issue.

Relatedly: don't let the unlocked/re-lock sequence (several seconds long) block nonce rotation.

#### relay attacks

This protocol does **not** defend against a [relay attack](https://en.wikipedia.org/wiki/Relay_attack) - two colluding devices, one near the phone and one near the lock, simply forwarding the BLE traffic between them in real time. This is the identical attack class that's been used against keyless-entry cars for years.

> **Q6.** Why doesn't the challenge-response scheme catch this? From the lock's perspective, what (if anything) looks different between a relayed exchange and a legitimately-in-range phone?

### Mobile webapp

The phone-side half of both [Pairing](#pairing) and [Unlock](#unlock): a single page, `index.html` (+ `mobile.css`, `mobile.js`, `ble_protocol.js`). There's no dashboard and nothing else in this webapp - this pairing/unlock page is the entire thing.

There is no new backend component here - no server, no database. Everything this page needs (the phone's keypair, its list of paired locks) lives entirely in the browser via IndexedDB. The lock itself is the only other party involved, spoken to directly over BLE.

#### Browser support - read this before you start debugging "nothing happens"

[Web Bluetooth](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API) is implemented by Chromium-based browsers (Chrome, Edge) on desktop and Android, and **not implemented at all** in Safari or Firefox. This isn't a temporary gap - it's a stance from Apple/WebKit and Mozilla, and it applies even to Chrome *on* iOS, because Apple's platform policy forces every browser on iOS to use WebKit under the hood regardless of what it's branded as. If you're testing on an iPhone, regular Chrome will not work here.

The practical workaround for iOS is a third-party browser that implements Web Bluetooth itself: **Bluefy - Web BLE Browser** (by Punch Through) is the commonly-used free option, available from the App Store. `index.html` already includes a `#no-bluetooth-banner` that should be shown (via feature-detecting `navigator.bluetooth`) when the current browser doesn't support this at all, so users land somewhere informative instead of a silent failure.

#### Testing locally, then hosting it

Don't just double-click `index.html` and open it as a `file://` URL - `<script type="module">` (which `index.html` uses to load `mobile.js`) is blocked from loading at all from a `file://` origin in most browsers, so nothing on the page will work: no locks ever load, buttons don't respond, and there's no error dialog to point at why, since the script itself never ran. Web Bluetooth has the same restriction more generally - it requires a [secure context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts) (HTTPS, or `localhost`).

Test locally first with:
```sh
python3 -m http.server 8000
```
from the webapp folder, then visit `http://localhost:8000` - `localhost` counts as a secure context, so this is enough to test on your own laptop's Bluetooth radio.

To actually use your phone as the controller, the page needs to be reachable from your phone's browser too, which means real HTTPS hosting. This project uses Firebase Hosting for that - the same tool part3 uses for its dashboard, so it's one deploy flow across both parts, and this is Hosting only, no Firestore/database involved:
- Go to the [Firebase console](https://console.firebase.google.com/) and create a new project. This also silently provisions a real Google Cloud project with the same ID behind it, whether or not you ever open the GCP console yourself - a Firebase project *is* a Google Cloud project with Firebase layered on top, not a separate thing. Remember which project you picked: part3 reuses this exact project (adding Firestore to it) instead of having you create a second one from scratch.
- In the sidebar, go to `Hosting & Serverless` -> `Hosting` -> `Get started`. This walks you straight through installing the CLI and logging in - no app registration or SDK config needed, since Hosting deploys at the project level rather than through a registered app (that registration step, under `Settings` -> `Your apps`, only exists to hand out a config for talking to *other* Firebase services like Firestore from your JS - not relevant here).
- Run the `npm install -g firebase-tools` and `firebase login` commands it shows you, but stop there - don't run the `firebase init`/`firebase deploy` it suggests next, since those need to run from your actual webapp folder below.
- From your webapp folder:
    - `firebase init hosting`
        - Provide `.` as the public directory
        - Select `Single-page app` as no
        - If asked whether to overwrite `index.html`, choose `No` - you already have one
    - `firebase deploy`

Firebase prints a `https://<project>.web.app` URL when it's done deploying - that's what you open on your phone. To take hosting down again: `firebase hosting:disable`.

#### Web Crypto: the phone's identity

The phone needs its own ECDSA P-256 keypair, generated once and reused for every lock it ever pairs with - not a fresh keypair per lock. This all happens through the browser's built-in [`SubtleCrypto`](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto) (`crypto.subtle`), no external crypto library needed:

- [`generateKey()`](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/generateKey) with `{ name: 'ECDSA', namedCurve: 'P-256' }` produces the pair.
- [`exportKey('raw', publicKey)`](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/exportKey) on the public key produces the same `0x04‖X‖Y` (65-byte) uncompressed point format `ecdsa_verify.py` expects on the device side - no format conversion needed.
- [`sign()`](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/sign) with `{ name: 'ECDSA', hash: 'SHA-256' }` over the private key produces a raw, concatenated `r‖s` signature (64 bytes for P-256) - again, exactly the format the device expects, no ASN.1/DER decoding required anywhere in this project.

Store the generated `CryptoKeyPair` object directly in IndexedDB (see below) - you don't need to serialize the private key yourself; the browser's structured-clone algorithm handles `CryptoKey` objects natively, while still enforcing non-extractability underneath.

#### IndexedDB: local, serverless storage

[IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) is the browser's built-in structured-storage database, and a reasonable place to put the two things this page needs to remember across visits: the phone's own identity (the keypair above), and the list of locks it's paired with (`lockId`, `name`, `deviceId`, plus whatever you need for reconnection - see below).

#### What's given vs. what you build

`mobile.js` only templates the four functions where the protocols from [Pairing](#pairing)/[Unlock](#unlock) actually get executed - `getOrCreateIdentity()`, `connectAndGetChars()`, `submitPairing()`, `unlock()` - since those are the parts that have to match the device byte-for-byte. Everything else - `ble_protocol.js` aside, which is the shared wire contract described above - is unwritten: how you structure the pairing UI, track per-lock connection state, reconnect, render `index.html`, wire it all together. None of that is graded on shape; build it however makes sense to you (an LLM can help here the same way it could with any other UI work).

#### "In range" and the auto-blur behavior

The unlock button is supposed to visually blur/disable itself when the phone isn't actively connected to the lock, and re-enable when it is. The natural-sounding approach - reading the BLE signal strength (RSSI) and blurring based on distance - isn't what this project uses, because the API that would expose that, [`watchAdvertisements()`](https://developer.mozilla.org/en-US/docs/Web/API/Bluetooth/watchAdvertisements), is still experimental and not reliably available across Web Bluetooth implementations.

Instead, "in range" here is judged from **GATT connection health** - a periodic read of `CHALLENGE` (or any characteristic) doubling as a liveness check: success means still connected, failure means flip the UI to "out of range." This state is purely cosmetic/advisory - the device is the actual source of truth for whether a nonce is still valid (see [Unlock](#unlock)), so a UI that's briefly wrong about "in range" doesn't create a security gap, only a UX one.

#### Rendering untrusted data

Everything this page displays that didn't originate as a hardcoded string in your own source - device names typed during pairing, status/error text - should be assumed to have come from an untrusted source by the time you're rendering it (a name is chosen by whoever is pairing, not validated by the browser). Build DOM structure normally, then assign user-influenced values via `textContent` (or element properties), never by interpolating them into an `innerHTML` string.

> **Q7.** Why does it matter whether you assign an untrusted string via `textContent` versus interpolating it into an `innerHTML` string? What could an attacker achieve by choosing a crafted device/display name, if this page ever rendered names that way?

### Threat modeling with STRIDE and MITRE ATT&CK

STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege), from lecture, categorizes *what kind* of threat you're looking at. [MITRE ATT&CK](https://attack.mitre.org/) answers a different question: *how would a real adversary actually pull it off*, via concrete **Techniques** (`T####`) grouped into **Tactics** (Initial Access, Credential Access, Impact, etc.). The two aren't redundant, and part of this exercise is noticing where ATT&CK's enterprise-network vocabulary doesn't cleanly cover a standalone embedded device.

> **Q8.** For each attack surface - [Pairing](#pairing), [Unlock](#unlock), [Mobile webapp](#mobile-webapp) - build a table with one row per STRIDE category: a concrete threat specific to that surface (not a restatement of the category), its mitigation in this design (cite where, or write "unmitigated"), and a matching [ATT&CK technique ID](https://attack.mitre.org/) (Mobile matrix for the webapp, Enterprise for Pairing/Unlock) - or "no technique fits" with one sentence why.
>
> Requirements: at least two rows across all three tables must be unmitigated, and at least one of those beyond the relay attack from Q6.

### Hand-In Procedure
You will turn in your assignments through Brightspace. The submission should be a zip file containing all the files that are required to run your code, a README.md explaining how to do it and the `answers.txt`.
