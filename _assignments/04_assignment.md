---
type: assignment
date: 2026-11-18T8:00:00-04:00
enable: yes
title: 'Lab #4 - SmartLock V3'
due_event: 
    type: due
    enable: yes
    date: 2026-12-11T23:59:00-04:00
    description: 'Lab #4 due'
---

# Remote updates

You can find the starter code for this part [here](https://github.com/purs3lab/ECE59500-130-iot/tree/part4). Submit an `answers.txt` file with answers to the descriptive questions that you will find throughout this part (Q1 to Q8).

Each time you want to add a new functionality to the system, you have to physically connect the device to your laptop and then flash it. To avoid this, we will add support for remote updates, in which a verified server can push security and feature updates to the device without physical access.

This part introduces a new role - a **server**, running on your laptop, that authors and signs updates - plus the device-side logic to receive, verify, and apply them safely, and to recover on its own if an update turns out to be broken. Because that's two fairly distinct problems, the instructions below are organized into sections:
- Setup, the new server role, and generating/installing its signing keypair - covered right below.
- [Remote updates](#remote-updates) - the update transport and protocol: how `server.py` publishes a signed update, and how the device verifies and applies it.
- [Automatic rollback](#automatic-rollback) - what happens when a pushed update is broken: crash-on-boot detection and automatic recovery via `boot.py`.

### Setup

**No new physical components, and no new MicroPython libraries.**

Note that `ecdsa_verify.py` is slightly different now. So make sure you copy this new version onto your pico.

`part4_starter_code/` is, again, *not* a self-contained project - build on your finished part3 solution, the same way part3 built on part1/part2:
- `common.py`, `main.py`, `init.py` in `part4_starter_code/` contain only the *new* pieces for this part, layered on top of your existing part3 code.
- `boot.py` is **brand new** - the crash-recovery guard MicroPython always runs before `main.py`. Covered in [Automatic rollback](#automatic-rollback).
- `keygen.py` is **brand new and given to you complete** - a one-time script that generates the server's update-signing keypair.
- `server.py` is **brand new** - the script you run from your laptop to publish a signed update.Covered in [Remote updates](#remote-updates).

**New laptop-side dependency.** `server.py`/`keygen.py` use the [`cryptography`](https://cryptography.io/) package to generate the server's keypair and sign updates:
```sh
pip install cryptography
```

### Architecture: "edge," "server," and "cloud"

- **`edge.py`** (part2) relays MQTT messages into Firestore. It has no real logic of its own, and **it doesn't change at all this part** - nothing about updates requires touching it.
- **Firestore/the cloud** is passive storage. It never runs logic - it just gets told about an `Action.UPDATE`/`Action.ROLLBACK` event the exact same way it learns about a `LOCK`/`UNLOCK` event, via the existing `edge.py` path, purely for dashboard visibility.
- **The server** (`server.py`, new this part) is different from both: it's the entity that actually holds the update-signing *private* key, decides what counts as a new "version," and authors/publishes signed update manifests and chunks. This runs logic (hashing, signing) and is the trust root the device relies on - so it earns the name "server" in a way `edge.py` never did. You'll likely run `edge.py` and `server.py` on the same laptop for convenience, but they're different components with different jobs.

### Generating and installing the server's signing keypair

Before you can publish anything, the server needs its own ECDSA P-256 keypair - separate from the phone's keypair from part3. Run once, from wherever you're keeping `server.py`:
```sh
python keygen.py
```
This produces two files:
- `server_private_key.pem` - **keep this secret.** It stays on your laptop and is never copied anywhere else. Anyone holding it can sign updates this lock will accept and install - treat it like a password, not a config file. (It's already covered by the starter repo's `.gitignore`, along with every other `.pem`/`.der` file - don't force-add it.)
- `server_pubkey.der` - the public half. **Copy this file onto your Pico's filesystem**, the same way you copied `ca.der` in part2. This is what `load_server_pubkey()` (see [Remote updates](#remote-updates)) reads to verify manifests against.

This copy has to happen over USB, not over the air.

> **Q1.** Why does the server's public key have to be installed this way, rather than the device just learning it from an MQTT message the first time it needs it?

### What you'll build

- **`server.py`**: a laptop script that takes a `.py` file, assigns it the next version number for that lock, hashes and signs it, and publishes the manifest + chunks to the update topic.
- **Device-side update handling**: reassembling chunks, verifying the manifest's signature and hash, rejecting stale/downgraded versions, and atomically swapping the new file into place. Full detail in [Remote updates](#remote-updates).
- **Crash-on-boot rollback**: if the newly applied file turns out broken, the device backs itself out to the last-known-good version on its own, without you ever touching it. Full detail in [Automatic rollback](#automatic-rollback).

### Remote updates

This covers publishing a signed update from your laptop (`server.py`), how the device receives/verifies/stages it, and how a paired controller triggers the actual install. What happens when an *applied* update turns out to be broken is a separate concern, covered in [Automatic rollback](#automatic-rollback).

"Update," here, means replacing a `.py` source file on the device's filesystem and rebooting - MicroPython is interpreted, so there's no firmware to reflash.

#### Transport

A new topic namespace, `secureiot/pico/update/{LOCKID}/...` - per-device, so one lock's update doesn't get applied to every lock listening on the broker. Updates need to be picked up promptly regardless of what the keypad is doing, so this part adds update handling into the same background task part3 already added for cloud pairing/unlock (`command_main()`), rather than a separate polling loop.

**Retained, per-artifact topics.** `server.py` publishes with `retain=True`, so the update sits "in the cloud" (on the broker) waiting, independent of whether the device happens to be online at push time - it'll get the full manifest+chunk set the moment it (re)subscribes, even if that's hours later. A broker only retains the *latest* message per topic. If the manifest and every chunk all published to one shared topic, retaining them would leave only the very last chunk recoverable - useless on its own. So each artifact gets its own topic instead:

- `secureiot/pico/update/{LOCKID}/manifest`
- `secureiot/pico/update/{LOCKID}/chunk/{seq}` (one topic per chunk index)

The device subscribes to the wildcard `secureiot/pico/update/{LOCKID}/#`, which the broker expands to "every retained message on every matching topic" on subscribe - manifest and all chunks arrive together, not just the newest one.

One consequence: since these can arrive as a batch of retained messages on a fresh subscribe, **the broker does not guarantee they arrive in `seq` order**. The device-side design below accounts for that.

#### Manifest and chunks

**Manifest** (published to the `/manifest` topic):
```json
{"type": "update_manifest", "target_file": "main.py", "version": 1, "sha256": "<hex>", "size": 4231, "chunk_size": 512, "num_chunks": 9, "signature": "<hex>"}
```

**What actually gets signed**
```
target_file_name (utf-8) || version (4 bytes, big-endian) || sha256 (32 raw bytes) || size (4 bytes, big-endian)
```
JSON serialization isn't guaranteed to produce byte-identical output between `server.py` (Python's `json`) and the device (MicroPython's `ujson`) - key order, whitespace, number formatting can all differ. Signing the JSON text itself would mean the device might reconstruct slightly different bytes than what was signed and fail verification for reasons that have nothing to do with security. The thing that actually needs to be trusted is the tuple of fields, not any particular encoding of them - so both sides independently build that exact byte string from the fields they have and check the signature against it. `signature` in the manifest is the usual raw `r || s` (64 bytes) - the same format `ecdsa_verify.verify()` already expects, just produced by `server.py`/`cryptography` on the laptop side instead of a phone's Web Crypto call.

**Chunks** (published to their own `/chunk/{seq}` topic):
```json
{"type": "update_chunk", "target_file": "main.py", "version": 1, "seq": 0, "data": "<base64>"}
```
Chunks are **individually unauthenticated** - there's no per-chunk signature. The device's only job with chunks themselves is reassembling them; the single manifest signature, checked once at the end, is the sole trust anchor for the whole update.

**`chunk_size` is not a fixed protocol value** - 512 is just a reasonable default that keeps each MQTT message small. Pick whatever you like; the only requirement is that `chunk_size`/`num_chunks` in the manifest stay internally consistent with however the file actually got split, on both ends.

- Track exactly one active update session at a time, keyed to the most recently *accepted* manifest (one that passed the manifest-level sanity checks below). A new valid manifest for a different `target_file`/`version` supersedes and discards whatever session was in progress; an identical manifest re-delivered (a retained redelivery, or a broker resend) should just be a no-op, not restart the session.
- Drop any chunk that doesn't match the active session's `target_file`/`version`, or whose `seq` falls outside `[0, num_chunks)`.
- Stage into a tmp file (e.g. `target_file + ".update.tmp"`) **pre-sized to the manifest's declared `size`**, and write each chunk at its own offset (`seq * chunk_size`) rather than appending sequentially - since retained delivery order isn't guaranteed, the device has to tolerate chunks arriving in any order, including a device that was offline for the whole original push and is now catching up in one burst.
- Reject a manifest whose `size` exceeds a sane cap for your device's free flash to begin with, and validate each chunk's length against what its slot should hold before writing it.

None of this is about proving *who* sent the update - only about making sure a hostile or confused publisher can waste, at worst, a bounded amount of flash and CPU before the device gives up and waits for something that actually verifies.

#### Staging: verify, but don't apply yet

Once a session's chunks are fully reassembled:

1. **Hash check.** Hash the staged file and compare it to the manifest's `sha256`.
2. **Signature check.** Reconstruct the canonical byte tuple from the manifest's own fields and verify it against the server's public key.
3. **Anti-downgrade check** - and this one specifically **only after the signature has passed**. Keep a small persisted record of what's currently installed, and require the new manifest's `version` to be strictly greater.

> **Q2.** Why must the anti-downgrade check run only *after* the signature check has already passed, never before?

If all three pass, the device does **not** install it yet. It persists a small "staged update" record (target file, version, hash) so this survives a reboot between now and whenever apply actually happens, and reports it with a dedicated `Action.UPDATE_STAGED` message (distinct from `Action.UPDATE`, which means "applied") - carrying the version/target_file/size as structured fields, not just a free-text reason. `edge.py` picks this up through the normal logging path (the same `secureiot/pico/data` topic every other action already uses) and sets `update_staged: true` (plus `staged_version`/`staged_target_file`) on the lock's Firestore doc, the same way it already sets `attested`. The dashboard shows a "ready to update" badge, and the cloud controller (`controller.html`) shows an **Apply Update** button on that lock's card.

> **Q3.** `edge.py` learns "this update is staged and verified" through the same normal `Action.UPDATE_STAGED` logging message every other action uses - not by reading the raw update manifest/chunk stream itself. Why does that distinction matter? How much can `edge.py` trust each of those two channels?

If any of the three checks fail, the device just discards the tmp file and reports the failure (still as `Action.UPDATE`, `success: false`) - nothing about a previously-staged, still-valid update is affected by an unrelated bad manifest turning up.

#### Applying: the controller has to say so

This is what makes the update "triggered by the controller" rather than self-triggered the instant it verifies. Applying reuses the **exact same signed challenge-response** the cloud controller already uses for unlock (see part3's Controller-via-cloud section) - `commands/{lockid}` -> `edge.py` -> `secureiot/pico/command/{LOCKID}` -> device issues a nonce -> controller signs it -> device verifies against the paired controller's key.

- The controller's initial request now carries an `intent` field: `"unlock"` or `"apply_update"`. The device stores it alongside the pending nonce and remembers it for when the signed response comes back.
- The bytes the controller signs use a new domain-separation tag, `TAG_APPLY_UPDATE`, instead of `TAG_CLOUD_UNLOCK` - same reasoning as every other tag split in this project: a signature captured for one action must never be replayable as authorization for a different one, even over the same channel.

Once verified, the device re-reads the staged file, re-checks its hash against the record made at staging time, re-checks the anti-downgrade condition, then does the atomic apply:

> **Q4.** The staged file's hash was already verified once, back when it finished staging, and both the staged file and its recorded hash (`staged_update.json`) sit as plain, unprotected files on the same on-device filesystem. Given that, what kind of *non-adversarial* issue between staging and applying is this re-check actually able to catch? And why doesn't it help against an attacker who already has write access to the device's local filesystem?

**Atomic apply.** Back up the current live file first (this is the start of [Automatic rollback](#automatic-rollback) - read that before implementing this step), rename the staged file over the live `target_file`, persist the new version number, send the final `Action.UPDATE` (`success: true`) with `qos=1` (this is the one message with the least room for error - block for the broker's PUBACK, since a plain `publish()` not raising doesn't guarantee it landed before `machine.reset()` a moment later tears the connection down), and reboot. Any authorization failure (unknown device, bad signature) or apply-time failure (nothing staged, hash no longer matches, stale version) is reported as `Action.UPDATE`, `success: false`, and the live file is never touched.

#### `server.py`

Given to you mostly complete - two pieces are left to fill in:
- **`last_push(lock_id, target_file)` / `save_push(...)`**: the server needs to hand out a strictly-increasing version number per `(lock, file)` pair across separate runs of the script (you'll run it again every time you push a new update), so it has to persist what it last handed out somewhere on disk - same idea as the old `next_version()`, just also remembering `num_chunks` from the last push.
- **The publish loop**: build and sign the manifest, then publish it (`retain=True`) to its own topic, then publish each chunk (`retain=True`) to its own topic. Before publishing, if this push has *fewer* chunks than the last one for this `(lock, file)`, clear the now-orphaned higher-`seq` chunk topics left over from the bigger push - publish an empty payload with `retain=True` to each; that's the only way to clear a topic's retained message. This isn't a correctness requirement (the device's version check already ignores stale chunks from an old push), just broker hygiene so retained state doesn't accumulate forever.

Run it as:
```sh
python server.py <LOCKID> <path/to/updated_file.py>
```
A good first test: take your working `main.py`, change one visible thing (an LCD string, say), push it, and confirm the dashboard/controller shows it as staged. Click **Apply Update** on the controller and watch the device install it and reset. Then try pushing something deliberately broken (a syntax error, an infinite loop) and apply *that*, to see [Automatic rollback](#automatic-rollback)'s recovery path in action.

### Automatic rollback

This covers what happens after a signed, verified, correctly-applied update turns out to be broken anyway - a typo, a logic bug, anything that keeps `main.py` from actually running.

#### Why `boot.py`

MicroPython always runs `boot.py` before `main.py`, on every single boot, unconditionally.

**`boot.py` itself must never be a target an update can touch.** This should be enforced on the device side wherever an incoming manifest's `target_file` gets validated (see [Remote updates](#remote-updates)).

> **Q5.** Why must `boot.py` be permanently off-limits to the update mechanism - what happens if it's the thing that ends up broken? And why does rejecting `"boot.py"` need to happen on the device side, rather than just trusting that `server.py` will never be pointed at it?

#### The mechanism

1. **Count boot attempts.** `boot.py` increments a persisted counter (e.g. `boot_state.json`, `{"attempts": n}`) at the very start of every boot - before anything from `main.py` has run.
2. **Clear the counter once `main.py` proves it's alive.** Somewhere early in `main()` reset that counter back to 0.
3. **Roll back once attempts exceed a threshold.** If `boot.py` finds the counter over some limit (e.g. 3) *and* a `.bak` exists: delete the broken `target_file`, rename `.bak` back to `target_file`, and reset the counter to 0.
4. **Report it, once you're back online.** `boot.py` itself has no MQTT access (it runs far too early, before WiFi/MQTT are even set up) - so it just leaves a marker (e.g. an empty `pending_rollback_report.flag` file) for `main.py` to notice once it's actually running again and has a working MQTT connection. `main.py` should check for that flag after connecting, and if present, remove it and send an `Action.ROLLBACK` event.

The `.bak` you restore in step 3 corresponds to an *older* version number than whatever was just installed and broke - but **restore only the file's bytes, not the version number**. Leave `installed_versions.json` claiming the crashed version is still "installed", even though the code that's actually running again is the older one.

> **Q6.** `server.py`'s update manifest+chunks sit on the broker as *retained* messages (see [Remote updates](#remote-updates)), so they're still there for the device to see on every future reboot, indefinitely, unless a newer push replaces them. Given that, what would go wrong on the very next boot if step 3 rolled `installed_versions.json` back to the older version number too, instead of leaving it claiming the crashed version?

Leaving the crashed version number in place costs nothing: a fix always ships as a strictly higher version anyway, since `server.py`'s own version counter only ever goes up.

An unhandled exception during `main.py`'s import or execution stops it before it reaches the "alive" checkpoint in step 3 - the counter increments, and after enough repeats, step 4 kicks in. That's most of what "a broken update" looks like in practice.

A **hang** - an infinite loop, a deadlock, anything that never actually crashes but also never returns - is different.

> **Q7.** Why can't the boot-attempt counter alone catch a hang the way it catches a crash? (Think about what has to happen for `boot.py` to run at all.) What does a **hardware watchdog** (`machine.WDT`) add that the counter can't provide on its own?

#### The watchdog (`machine.WDT`)

`machine.WDT` is a hardware peripheral: once armed with a timeout, the chip force-resets itself unless something calls `.feed()` within that window. A forced reset re-runs `boot.py` from scratch, which is exactly the trigger the crash-counter mechanism above needs - so the watchdog turns a hang into the same recoverable case as a crash.

- **It can't be disabled once armed, only reset.** That makes it actively hostile to iterating in the REPL - a breakpoint or a slow manual step would force-reset the board out from under you. Gate arming it behind a flag file you control (e.g. `enable_wdt.flag`), so it's off by default while you're still developing and you turn it on once the rest of the flow is working. `boot.py` should be what actually arms the peripheral (`machine.WDT(timeout=...)`) - since it runs before anything in `main.py` could fail, even a syntax error in `main.py` still gets caught by a watchdog that was already armed. `main.py` then only needs a *handle* to that same peripheral to feed it (calling `machine.WDT(...)` again doesn't create a second timer).
- **Slow synchronous work needs to feed it too.** Signature verification (see `ecdsa_verify.verify()` - introduced back in part3, used again here in [Remote updates](#remote-updates)) does a full elliptic-curve scalar multiplication, which is slow enough on its own to blow past the timeout with nothing else able to run in the meantime - a single long synchronous call can starve the task that would otherwise be feeding the watchdog. Whatever calls `verify()` during update handling needs to keep feeding the watchdog through that specific call, not just from its usual background loop.

If you suddenly see the board rebooting without any reason, the most likely culprit is the watchdog.

### Threat modeling with STRIDE and MITRE ATT&CK

In part2 (Q8) you did STRIDE + ATT&CK by hand for the BLE surfaces, and in part3 you modeled the cloud path with Threagile. This part adds a new kind of trust: the device now accepts and runs *code* it didn't ship with. So this time, go back to doing it by hand, focused only on the update path.

> **Q8.** For each attack surface - [Update delivery](#manifest-and-chunks) (`server.py` -> broker -> device: manifest, chunks, staging), [Apply trigger](#applying-the-controller-has-to-say-so) (controller -> `edge.py` -> device challenge-response), and [Automatic rollback](#automatic-rollback) (`boot.py`, boot counter, watchdog) - build a table with one row per STRIDE category: a concrete threat specific to that surface (not a restatement of the category), its mitigation in this design (cite where, or write "unmitigated"), and a matching [ATT&CK technique ID](https://attack.mitre.org/) - or "no technique fits" with one sentence why. You may use the Enterprise or the [ICS](https://attack.mitre.org/matrices/ics/) matrix; say which one for each ID.
>
> Requirements: at least two rows across all three tables must be unmitigated, and at least one row must involve the server's signing key (`server_private_key.pem`).

### Hand-In Procedure
> Note: This part also requires you to upload a video demonstrating your implementation. For each operation, the video must show both the webpage and the Pico. A single camera is fine: start on the device where the operation is initiated (the webpage or the Pico's buttons), perform it, then point the camera at the other device to show the result.

You will turn in your assignments through Brightspace. The submission should be a zip file containing all the files that are required to run your code, a README.md explaining how to do it, the demo video and the `answers.txt`.
