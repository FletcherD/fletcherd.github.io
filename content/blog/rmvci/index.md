+++
title = "Teaching Linux to talk to my Prius: reverse-engineering Techstream and the Mini-VCI cable"
description = "How a question about one A/C servo turned into a cable firmware teardown, a from-scratch Rust J2534 driver, a decoded dealer diagnostic database, and a first open release. Part one."
date = 2026-10-16

[taxonomies]
tags = ["reverse engineering", "Rust", "automotive", "firmware", "Ghidra"]

[extra]
toc = true
+++

*How one number from my car's climate control turned into a firmware teardown of a cheap diagnostic cable, a from-scratch Rust driver, a decoded dealer database covering 2,480 vehicles, and Toyota's own diagnostic engine running headless under Wine. Part one.*

**[rmvci on GitHub](https://github.com/FletcherD/rmvci)** • **[rmvci-core 0.1.0 release](https://github.com/FletcherD/rmvci/releases/tag/v0.1.0)**

---

Toyota's dealer diagnostic software is called **Techstream**. It's how a technician reads live data, trouble codes, and actuator tests from every computer in a Toyota, and it's a 32-bit Windows application that people keep alive in virtual machines. The cable it talks through is a **J2534 pass-through** adapter. The cheap clone everyone uses, the **Mini-VCI**, costs about the price of lunch and has exactly one Linux driver, which only speaks half of what the cable can do.

I wanted to read a single value from my Gen-2 (NHW20) Prius on Linux: the position of the **air-mix blend door servo** in the climate control, commanded versus actual. This post is how that turned into the whole project, roughly **August 6 to 12, 2026**, plus the release I cut today. Like my other write-ups, it's chronological and includes the dead ends, because the dead ends are where most of the learning was.

---

## Act I: The value isn't in any database

The plan was supposed to be short. **OBDb** is an open project with per-vehicle JSON databases of diagnostic signals. If the Prius repo had the climate servo, this would be a config file and I'd be done.

It didn't. `OBDb/Toyota-Prius` covers the powertrain, hybrid battery, and chassis, and never talks to the A/C amplifier at all. The shared Toyota base repo did have a climate block, used by the Camry, RX, and 4Runner, on CAN ID `7C4` with a KWP2000 read `21 43`, where byte 0 is the target and byte 1 the actual servo position. Promising, but unverified on a Prius. (That turned out to matter a lot.)

To send `21 43` to `7C4` I needed a way onto the car's CAN bus. I had the Mini-VCI cable. Its only Linux driver, **[libMVCI](https://github.com/aselafernando/libMVCI)** by Asela Fernando, supports K-line but not CAN: "CAN ECUs are not yet reachable." So the next question was why not, and whether the cable could do it.

---

## Act II: Reading the cable's mind

The cable is an FTDI FT232R USB-serial chip feeding an NXP LPC2119 microcontroller running firmware that identifies itself as **"J2534 MINI V1.03"**. The Windows driver DLL is protected by a commercial packer: its import table is a three-function decoy and its code section is encrypted at rest, so disassembling `PassThruOpen` gives you noise.

The firmware, though, was an easy target. A public firmware-restore image for this cable has **code read protection switched off**, and Ghidra pulled 480 functions out of it: an ARM bootloader at `0x0` and the J2534 application at `0x8000`. Live probes on my cable (no car attached) settled the first question immediately. The firmware accepts protocol IDs 1 through 6, which include CAN and ISO 15765. **CAN works in the cable; the gap was entirely in the driver.**

I could have just added CAN to libMVCI then and there. I chose to reverse the firmware properly first, and I'm glad I did, because this cable has traps.

### The traps

Every one of these cost real time and is now encoded in the driver's API:

- **Silence until you install a filter.** The CAN acceptance filter comes up *enabled and empty*. A channel with no filter receives nothing at all, which looks exactly like a dead bus. (There's a filter-bypass function in the firmware. Nothing calls it.)
- **The mask isn't a mask.** The firmware programs a *range*, `[pattern & mask, pattern | ~mask]`. So the J2534-habitual `0x7FF` mask matches nothing useful, and matching one exact CAN ID needs `FF FF FF FF`.
- **One filter per ID width.** Installing a filter silently replaces the previous one.
- **`SET_CONFIG` lies.** Unknown parameters return success and are thrown away. All the ISO-TP flow-control settings (block size, separation time, padding) are stored and never read.
- **Protocol IDs 7–11 crash the MCU.** The firmware indexes a six-entry table with `id - 1` and checks only for zero, so the empty slots (filled with `0x55555555`) make it jump into garbage. Recovery needs a port reopen, which pulses DTR/RTS to reset the chip.
- **The firmware's ISO-TP is shaky.** It ignores the receiver's block size, uses a fixed 1 ms separation time, and truncates the first-frame length to 8 bits, so messages over 255 bytes break. This also explains a bit of folklore about a "~48-byte limit" with these cables: 6 + 6×7 = 48 is just an ECU advertising a block size of 6 and giving up when the cable overruns it.

I also found four bugs in libMVCI's protocol notes. Lengths are 16-bit, not 8. The status byte is at offset 3, not the end. The trailing bytes are uninitialized stack. And libMVCI's "disconnect" command is actually the one that tears down the encrypted session, which explains its re-handshake-after-disconnect workaround. Some of these I could prove from libMVCI's own captured test vectors, without touching the decompiled firmware at all.

### The bench

With no car on my desk, I wired the cable to a cheap CH340 USB-CAN analyzer and wrote a fake ECU in Python to answer `21 43`. The first evening I chased a "broken transmit path" for a while before realizing the broken thing was my instrumentation: block-buffered stdout lost on a timeout, and a leftover monitor process holding the serial port. Once that was sorted, the response came back: `61 43 7b 79`, commanded 123, actual 121, from my fake servo.

The best moment of the bench work was the flow-control frame. The cable answered my fake ECU's multi-frame response with `30 00 01`, the exact hard-coded bytes I'd read out of the disassembly an hour earlier. A prediction from a decompiler, confirmed on a wire.

---

## Act III: rMVCI, a new driver in a day

My original plan was to contribute CAN support back to libMVCI as a series of PRs. I had a working branch: CAN, ISO-TP, the protocol fixes, 24 of 24 test vectors passing, and no K-line regressions. But the more I designed, the more I wanted things the C code couldn't easily give me. So on **August 7** I wrote a new driver in Rust, called **rMVCI**. (The CAN work on libMVCI is still sitting on a local branch. I never upstreamed it, and that's on my list.)

The design came out of the traps above:

- A **sans-IO codec**, so all of the wire protocol is testable without hardware. libMVCI's 24 captured test vectors were ported byte-for-byte.
- A **transport trait** with serial, direct-USB, and mock backends, and one **actor thread** that owns the port, instead of a mutex.
- **Typed channels**, `Channel<P>`, where the protocol is a type, so a filter of the wrong width can't be expressed and no out-of-range protocol ID can reach the wire.
- **ISO-TP done two ways:** through the firmware (fast, but with all its limits) or **host-side over raw CAN** (one USB round trip per frame, but correct block size, separation time, and long messages). One type, one argument to pick the path.
- A `cdylib` exporting the 14 standard J2534 `PassThru*` functions, so Windows tools like Techstream can load it.

The milestones landed between 10:25 and 11:06 that morning, with fuzz targets running about 35 million executions without a crash. The demo program decoded "commanded 123, actual 121" from the bench ECU on both ISO-TP paths.

### The keepalive that wasn't needed

libMVCI pings the cable every 15 ms because "the adapter resets if left idle". My first test agreed the session survived five minutes idle, but it only asked for the firmware version afterward, which proves the device is alive, not that your channel still works. So I made it a staged test: after each idle period, check the session, then the channel, then a full request and response.

| Idle | Session | Channel | Exchange |
|---|---|---|---|
| 5 s | ok | ok | ok |
| 60 s | ok | ok | ok |
| 300 s | ok | ok | ok |
| **900 s** | **ok** | **ok** | **ok** |

Fifteen minutes of silence and everything still works. rMVCI sends nothing when it has nothing to say, and detects a wedged adapter by counting unanswered *real* requests instead. (The C keepalive was also silently discarding received data.)

The same day I measured the two ISO-TP paths against a fake ECU that asked for a block size of 2 and a 50 ms separation time. The firmware path sent 7 frames with roughly 0 ms gaps. The host path paused after exactly 2, at 50.0 ms.

### Android

My real goal at this point was an app like Techstream on my phone. Plugging the cable into a rooted Android phone flushed out four bugs in the JNI layer on the first run:

1. USB permission always came back denied, because an immutable `PendingIntent` stops Android from attaching the "granted" extra.
2. The USB serial library rejects reads smaller than the 64-byte packet size, and the driver was reading 2-byte headers.
3. The app was killed (SIGKILL) by an uncleared Java exception on the actor thread, from a write racing the port close.
4. Closing took 5 seconds because a cached ISO-TP object held a second handle to the actor, so the actor never saw the session end. Fixed, close dropped to 17 ms.

Then the phone read the bench ECU through the cable: commanded 123, actual 121. I also added a direct-USB backend on Linux using `nusb`, which can set the FTDI latency timer without root and cut exchange time from 48 ms to 30 ms.

---

## Act IV: Techstream is a database with an engine attached

With the cable sorted, the question became *what to send*. My one known request, `21 43` on CAN `7C4`, still wasn't verified for my car. Techstream knows the answer for every Toyota, so I started taking it apart.

### It won't run on Linux

I spent most of an evening trying to run Techstream under Wine, purely to see it. Its data lives in Microsoft Access databases behind ADO, which meant hand-installing the Jet database engine, finding that its OLE DB provider is renamed at install time by an INF file, copying native ADO DLLs over Wine's placeholders, and bumping the program's stack reserve in its PE header from 1 MB to 16 MB to stop a stack overflow. Every version I tried, from v7 to v18, then died the same way: an uncaught C++ exception. Much later, with a debugger working, I traced it to Wine's 32-bit C++ exception handling, where an exception thrown from inside a `catch` block loses the enclosing handler.

So I gave up and built a Windows XP virtual machine, with the cable passed through over USB. Techstream came up immediately and happily built its menus for my car, offline.

### The data model

The single most important finding of the whole project came next. **Techstream contains almost no car knowledge.** It's a fairly small generic engine plus a database:

- `.vds` files are renamed Access databases with vehicle lists and ECU addressing.
- About 437 `.ddb` files, one per ECU family, hold every signal, trouble code, actuator test, and customization setting, *including the literal request bytes to send* and the byte and bit slice to decode from the answer.
- All the names live in a separate string table: 183,244 strings, UTF-16, LZSS-compressed. (Grepping for "Air Mix" found nothing until the decompressor worked.)

This is what makes an open replacement possible. You don't reverse-engineer thousands of vehicles. You reverse-engineer one database format, once.

The `.ddb` families turned out to be protocol generations: P3 ECUs use KWP2000 `21 <LID>` reads, P5 ECUs use UDS `22 <DID>`, P4 is a mix. Matching trouble codes identified `A_C_P3.ddb` as my car's A/C amplifier, which meant the `21 43` from OBDb belonged to a *newer* generation. My target request had been wrong the whole time.

### Decoding it properly

Getting from "this is a database" to correct output took many rounds, and the theme of all of them was refusing to guess:

- **String IDs** were being masked to 16 bits. ABS uses IDs around 95,000. That bug turned up three separate times in different fields. At one point a decoder "fixed" bad names with a regex filter, and I lost my patience: *"that CANNOT be what techstream actually does."* The real fix brought bad names from 6,124 down to 9.
- **Scaling** is `(raw × num / den + offset) / 10^decimals`. The ambient temperature sensor came out as `×0.35 − 23.3 °C`, exactly what OBDb has for it.
- **Checking against OBDb** as outside ground truth: A/C names matched 12 of 12, and the P5 generation about 99%.
- **The vehicle → ECU map** for all 2,480 vehicles came from the database's menu tables. When a guessed vehicle make crept into the extractor, it came straight back out: no guesses anywhere.
- **Which bus an ECU is on** is decided by one record: an address of `0xFF` or less means K-line, larger means CAN. The generation suffix only picks the decoder.

That last rule matters here, because it says my car's A/C amplifier isn't on CAN at all.

### Watching Techstream work

Static analysis gets you the database. To see what Techstream *sends*, I put a logging layer at the J2534 boundary in the VM: a small DLL with the 14 standard `PassThru*` entry points that records every call, including the raw request bytes, and streams the log to my Linux host. (The first build crashed Techstream on launch, because it opened a network socket from `DllMain`, which deadlocks under the Windows loader lock. Lazy init fixed it.)

The first trace was worth the whole exercise. When I selected my car and opened the Air Conditioner screen, Techstream never touched CAN `7C4`. It switched to **K-line, ISO 14230, address `0x98`**, exactly what the database rule had predicted, sent a few identification reads, then polled `21 E1` hundreds of times. Then I drove the VM with OCR-gated automation and captured request frames from 11 of the 15 ECUs in the car.

Two cautions came out of this. First, the logger was *faking* ECU responses, so it could show me real requests but nothing real coming back. Second, I over-read the capture: I decided `21 E1` was a bulk "read the whole data list" command. It isn't. On the real car it returns a two-byte trouble-code count, and the per-LID reads in the database are what Techstream actually uses. A mock can tell you what a program asks, never what the world answers.

---

## Act V: First contact

On **August 9**, after three days of bench work, the cable finally went into my car. It sat in a Raspberry Pi in the driveway running `rmvci-bridge`, a small TCP service that forwards the driver's five transport primitives, so I could drive it from my desk.

The demo program would have been wrong on the first byte: it still sent CAN `7C4`. I wrote a K-line client instead. The first attempt failed outright. FAST_INIT returned a single status byte, `08`, and then every write timed out at the adapter. I'd skipped the vendor bring-up sequence: twelve `SET_CONFIG` timing parameters that Techstream sets before touching K-line, plus a periodic-message clear before init and before each write. Those "magic numbers" turned out to be ordinary J2534 timing IDs (9600 baud, `P1_MAX`, `TINIL`, …) with Toyota's captured values.

Second attempt: FAST_INIT came back with the full key bytes, `83 f0 98 c1 e9 8f 44`, exactly as predicted. But the data was off by one. Every reply seemed to belong to the *previous* request. The A/C amplifier answers `7f 21 78` first, which is "response pending", and sends the real `61 <LID>` a moment later. Once the driver drained the pending frames, at about 3 PM:

| Signal | Request | Raw | Value |
|---|---|---|---|
| Air-mix damper, actual | `21 0D` | 9 | **−9.5 %** |
| Air-mix damper, target | `21 13` | 8 | **−10.0 %** |
| Ambient temperature | `21 02` | 164 | 34.1 °C |
| Blower | `21 1C` | 0 | off |

The servo I'd started this whole project to read was within half a percent of where it had been told to go.

### Validating the whole car, read-only

That evening I ran the reverse-engineered vehicle map against every ECU in the car, using only reads: no writes, no tests, no code clears.

- **13 of 13 ECUs** answered at the bus and address the database predicted. Eight on K-line (A/C, ABS, body, steering, gateway, ignition, immobiliser, transmission), three on CAN (engine, hybrid, hybrid battery). The key-byte format held for every one.
- The A/C values made physical sense for a car parked in the sun: ambient 33 °C, cabin 39 °C, coolant 62.6 °C.
- The engine ECU reported a stored **P0420**, catalyst efficiency, which is practically a rite of passage for an aging Prius. A 50-byte multi-frame ISO-TP response from the hybrid ECU reassembled correctly.
- Two LIDs on the A/C amp (`21 52` and the DTC read) produced an endless storm of "response pending" frames and then left the amplifier wedged until re-initialised. The driver now drains the storm and re-inits automatically. The extraction later explained it: `21 52` is the *one* A/C LID the database flags as unsupported on this ECU.
- Five chassis ECUs answered their identification read but stayed silent on live data. My first theory was that they needed a diagnostic session started first. Wrong, as the next day showed.

### Techstream, through my driver, to my car

The next step was to turn the logging DLL into a *forwarding* one. Instead of faking replies, it sent each J2534 call to a small server on my PC that replayed it on rMVCI, over TCP, to the Pi, to the cable, to the car. Techstream in the VM, talking to a real Prius through a Rust driver it had never heard of.

*"it worked!! it connected to the car and showed me real data."* 686 real ECU responses in that session, including 65 from the ABS and 51 from the body ECU, the ones my own tool couldn't read. The capture showed why: when a LID goes unanswered, Techstream re-runs FAST_INIT after two empty reads and carries on. My driver didn't. Now it does. That one capture also corrected an error in my static analysis: the KWP trouble-code read is service `0x13`, not `0x18`, proven on the ABS with 13 codes that matched its own `21 E1` count. Two neighbouring commands found by the same method are now flagged as suspect until I see them on a wire.

A later full-vehicle capture, with me clicking through every ECU and data list in the GUI, covered 15 K-line and 5 CAN ECUs with real responses, and showed how the CAN-side data lists work: `A8 01` enumerates the PIDs, `A1` defines a poll set, `A2` polls it. Active tests are service `30`. Customisation writes never appeared at all, which explained why that screen did nothing.

One more theory died to measurement. Techstream v18 hung on connect after asking for 10400 baud on K-line, and I blamed the clone cable. So I sniffed the *genuine* Windows driver talking to the cable (USB capture, deframe, decrypt the session) and compared it to mine: byte-identical. The real Techstream just tries 10400, gets refused, and falls back to 9600 on its own.

---

## Act VI: The engine without the window

Driving a Windows GUI inside a VM to produce ground truth is miserable, so I wanted Techstream's diagnostic engine without Techstream's window. The engine is a chain of DLLs behind a `DiagAPI` facade, and the crash that kills the GUI under Wine lives in code that chain never calls. So I built a harness that loads the engine DLLs under Wine, with the J2534 layer answered by a **replay oracle** built from my car captures. No GUI, no VM, no car.

Getting it to "finalize" a vehicle took a day of finding undocumented preconditions (a Smart Key option word, commit IDs that were dimension IDs rather than values). Once it did, it read the VIN over the replay, identified "NHW20 1NZFXE", and emitted request frames that matched the real capture **byte for byte**: 98 of 98 for the A/C amplifier, and enumerated data lists for six of the nine K-line ECUs. The extractor's output turned out to be a strict superset of what the engine lists for each of them.

Then I stopped. The harness enters the live-value poll path correctly but fails a precondition check before any `21 <LID>` goes out, and I couldn't find the cause in the time I had. It's on the list below.

### Writing down what I actually knew

By August 10 I was, in my own words to the agent, *"kind of overwhelmed with both the complexity of all the message types and edge cases, and the question of how to even model this system."* Findings were spread across forty-odd markdown files, some contradicting each other. So I spent a day on nothing but consolidation: every document got a status banner (current, log, or superseded), a single `DATA_MODEL.md` became the canonical synthesis, and an index that holds no findings of its own points at the rest.

The extraction scripts became a real pipeline, `techstream_extract`. It runs against a Techstream installation and, in about four seconds, produces, for all **415 ECUs and 2,480 vehicles**: 56,684 data-list signals, 61,072 trouble codes, 5,995 active tests, 3,451 freeze-frame items, 2,139 customisation settings, and the vehicle-to-ECU map, each request tagged with where in the database it came from. The tool is published. The data it produces is not, because it derives from Toyota's database; you run it against your own installation.

Rewriting the extractor as a pipeline found bugs the one-off scripts had hidden. The field I'd been calling the "mask" is a *validity* mask, and using it to extract values would have silently truncated 23% of bit-sliced signals; it never showed up on the car because I'd only checked byte-aligned ones. Trouble codes aren't unique within an ECU (one ABS code appears four times with different discriminators). Analog active tests take two-byte drive values after all.

---

## Act VII: Shipping something

On August 30 the rMVCI repo went public, after the two ISO-TP implementations were merged into one type with a path argument. Today, **October 16**, I cut **rmvci-core 0.1.0** on GitHub: CI on Linux, macOS, Windows, and Android, a minimum Rust version that CI actually checks (it caught that the declared one was wrong), and a README whose Prius example finally shows the K-line read that works instead of the CAN read that doesn't. Adding CI found three real bugs before anyone else could.

It's not on crates.io yet (no login on the build machine, as mundane as that). It's GPL-3.0, because it descends from libMVCI's captured vectors and I haven't asked the author about relicensing. And it's just the core: the J2534 shared library and the Android layer are in the repo as source, while `rmvci-diag` (KWP2000 and UDS clients) and `rmvci-net` (the TCP bridge) are held back, because diag doesn't yet speak the DTC service I only learned about on the car, and the bridge listens on all interfaces with no authentication.

---

## What remains

This is part one because the project isn't done. In rough priority order:

1. **Publish to crates.io**, core first, then diag, then net.
2. **Finish `rmvci-diag`:** DTC read (`13`) and clear (`14`), freeze frames, active tests, then customisation writes behind a confirmation step. Turn the car captures into mock tests so none of this needs a driveway. Mark anything unverified as such.
3. **Make the bridge safe to publish:** bind to localhost by default, or require a flag.
4. **`tsdiag`**, a first cut at the data-driven CLI (scan the car, decode everything using the extracted JSON), has been written but never run against the car.
5. **The harness value poll:** find the unmet precondition so the engine emits `21 <LID>` polls headless, then the P4/P5 data-list path (`A8 01`), which it can't do yet.
6. **Extraction gaps:** CAN-side scale factors are overridden by two tables I haven't decoded (the largest known correctness gap); DTC clear has never been exercised; two still-undecoded database fields are catalogued with what's been ruled out.
7. **Things only a wire can settle:** actuator-drive frames have never been captured because I've never fired a test; security access (`27`) has never been seen, so the immobiliser, theft, and meter ECUs return nothing.
8. **Ideas on the shelf:** a vendor-driver-versus-rMVCI conformance corpus, Windows testing of the J2534 library, and that libMVCI branch I owe upstream.
9. **Relicense to MIT**, if the libMVCI author agrees.

---

## What I learned

**Mocks tell you what a program asks, never what the world answers.** The `21 E1` theory, the "silent ECUs need a session" theory, and the "clone cable can't do 10400" theory all came from captures with fake responses, and all three died the moment a real ECU or a real driver was in the loop.

**Reverse the thing with the least protection.** The Windows DLL was armoured. The cable firmware wasn't, and it told me everything the DLL would have, including a hard-coded flow-control frame I then saw on the wire.

**No guesses.** Every time a guess crept into the extractor, a regex that filtered "bad" names, a hand-picked vehicle make, a decoder chosen by filename, it was wrong, and worse, it was wrong quietly. The fixes were always to go back to the decompilation and find the real rule.

**Measure the claims you inherit.** "The adapter resets if idle" cost the previous driver a 15 ms polling loop forever. Fifteen minutes of silence says otherwise.

**When you're overwhelmed, stop and write.** The consolidation day produced no new findings and was the most productive day of the project.

Everything here was done on my own car, with my own cable, and the only writes to the vehicle were the ones Techstream itself made. The driver is at **[github.com/FletcherD/rmvci](https://github.com/FletcherD/rmvci)**. Part two will be the diagnostic layer, the CLI, and whatever the car refutes next.