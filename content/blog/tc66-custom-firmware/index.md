+++
title = "Reverse-engineering a $30 USB power meter, and writing it a new brain"
description = "How I went from \"why won't this meter do what I want over USB\" to a from-scratch replacement firmware that triggers USB-PD, QuickCharge, and Samsung AFC on command."
date = 2026-09-21
updated = 2026-10-16

[taxonomies]
tags = ["reverse engineering", "Rust", "firmware", "USB-PD", "hardware"]

[extra]
toc = true
+++

*How I went from "why won't this meter do what I want over USB" to a from-scratch replacement firmware that triggers USB-PD, QuickCharge, and Samsung AFC on command — by way of a failed side-channel attack, a live SRAM heist, a full Ghidra teardown of the vendor firmware, and a microscope.*

---

The RuiDeng TC66C is a lovely little thing: a USB-C inline power meter with a colour screen that reports voltage, current, power, and cumulative energy for whatever you plug through it. It costs about the price of a large pizza. I bought one to profile the idle-vs-active power draw of a recorder board I was working on, and for that it's perfect.

Then I wanted to do something slightly more ambitious — I wanted to *script* it. Poll it fast, log cleanly, and (this is the part that turned into a rabbit hole eleven days deep) make it trigger a charger up to 9 V or 12 V on command from my PC, the way it can from its own buttons. The meter's box literally calls it a "Type-C USB PD Trigger." Surely that's reachable over the wire.

It is not. And chasing *why not* took me through essentially every layer of the device — its host software, its wire protocol, its encrypted firmware, the readout-protected chip that firmware runs on, the physical PCB under a microscope, and finally a complete clean-room replacement firmware written in Rust. This is the whole story, dead ends included. Especially the dead ends.

**The firmware is open source (MIT): [github.com/FletcherD/tc66-custom-fw](https://github.com/FletcherD/tc66-custom-fw)** — prebuilt releases, a one-command flashing tool, and a path back to stock.

A note on the timeline before we start: this happened fast and dense — roughly **August 15–26, 2026**. The reverse-engineering and the attacks on the chip filled the first week; the from-scratch firmware then landed in a burst (13 commits one day, 20 the next, 32 the day after that). I've kept things chronological, but a few threads ran in parallel and I'll flag it when they did. The firmware kept growing afterwards, in shorter bursts through September and mid-October; where that later work belongs to a topic below, it's folded into that topic's section.

---

## Act I — The protocol, and the word that wasn't there

The TC66C shows up on Linux as a plain USB CDC serial port (`/dev/ttyACM*`). No driver needed. So the first question is simply: what do you *say* to it?

There are two pieces of official software to learn from — a Windows PC app and an Android app — and neither hands you the answer politely.

### Cracking open the PC app

The Windows app, `RuiDengUSBMeter.exe` v1.0.0.6, is a .NET application. Two obstacles: its dependencies are bundled inside the executable with **Costura** (deflate-compressed into resources), and it's lightly obfuscated with **ConfuserEx**, which *encrypts constant strings*. That second detail matters more than it sounds: run `strings` on the binary and you find **nothing** — no command verbs, no protocol. A naive look would conclude the protocol was hidden. It isn't; it's just encrypted at rest and decrypted at runtime.

The move is to decompile with ILSpy (`ilspycmd -p -o out RuiDengUSBMeter.exe`), which resolves ConfuserEx's string encryption automatically and gives you clean, decrypted C#. And the entire serial surface of the app collapses to a single class — `MainWindow.cs` — which holds the one and only `SerialPort` in the whole program. Every `serial.Write` in the app either sends one of a handful of fixed 5-character strings, or streams a firmware image in 64-byte chunks. I extracted all 13 Costura-embedded assemblies to be thorough; they're stock open-source libraries (JSON, Excel export, WPF charting) and none of them touch the serial port. There was even a trap along the way: substring hits for "clear" and "record" that turned out to be ordinary English inside XML strings, not commands.

### The Android app fought back harder

The Android app (`com.rdtc.blesample`) is **packed with 360 Jiagu**. The `classes.dex` in the APK is a six-class stub; the real code is decrypted into memory at runtime, so static analysis of the file gives you nothing at all. Worse, 360 Jiagu **wipes the 8-byte DEX magic in the live memory mapping** after the runtime has loaded it — an explicit anti-dump measure — so you can't even find the real DEX by scanning for its header.

The way through, on a rooted phone with the app running:

1. Read `/proc/<pid>/maps` as root.
2. Find the real DEX not by its (wiped) magic but **by size** — it's the anonymous readable region whose length matches the DEX's declared `file_size` (here ~4.5 MB, 3480 classes).
3. `dd` that region out of `/proc/<pid>/mem`.
4. **Repair** it by writing `dex\n035\0` back into bytes 0–7. The stored Adler-32 checksum (which covers bytes 12+, not the magic) already matched — proof the body was intact — then recompute the SHA-1 signature to make it canonical.

Then `baksmali` and `jadx` on the repaired DEX. The payoff: the Android app is **BLE-only** — it has no USB code path whatsoever — and its command set is a thin mirror of the USB one.

### The complete command set

Between the two apps, the entire vocabulary of the TC66C is seven bare, 5-byte, unterminated ASCII commands:

| Command | Reply | Purpose |
|---|---|---|
| `getva` | 192 bytes, encrypted | poll all measurements |
| `rotat` | *(none)* | rotate the screen 180° |
| `nextp` / `lastp` | *(none)* | next / previous on-device screen |
| `gtrec` | 8-byte records + `reck` | dump the internal ~1 Hz datalogger |
| `query` | `firm…` or `boot` | ask whether the app or bootloader is running |
| `update` | `uprdy`, then `OK`/chunk | (in the bootloader) begin flashing |

Over BLE it's the same verbs with a `b` prefix and a CRLF (`bgetva\r\n`, etc.). That's *it*. And crucially, note what is **not** there: no command to start/stop/clear recording, no group reset, no calibration, and — the one I came for — **no PD/PPS trigger, no QuickCharge, no Samsung AFC, no Huawei FCP/SCP**. I swept both apps exhaustively for any byte sequence, hex literal, or base64 blob that could be a hidden command. Nothing. The fast-charge trigger is real in the device — the vendor changelog even dates it to firmware v1.11 in 2019 — but it's driven entirely from the physical buttons. **The wire protocol simply does not expose it.**

That negative result is the hinge of this whole project. If the trigger had been one `serial.Write` away, I'd have written a 20-line script and gone to bed. Because it wasn't, the only way down was *through* — into the firmware and the silicon.

### The details worth keeping

A few things I learned characterising the protocol properly, because they shaped everything later:

**The AES key gotcha.** The `getva` reply is 192 bytes = three 64-byte blocks (`pac1`/`pac2`/`pac3`), AES-256-ECB encrypted with a static factory key, with measurements at fixed little-endian offsets after you decrypt (voltage /10000 → V at offset 48, current /100000 → A at 52, and so on). Two keys for this circulate online, and **only one of them works on my unit** — the so-called "rdserialtool" key (`5821fa56 01b2f026 …`). The other, widely-posted "decimal" key decrypts to pure garbage. If you'd started from the wrong reference you'd swear the format was undocumented. (This exact key later turned up sitting in the firmware image, confirming it.)

**The parser is dumber than you'd think, in a useful way.** I wrote a probe harness (`protocol_probe.py`) that drives both of my meters at once and fuzzes the framing. One rule explains all the behaviour: *the firmware reads one USB packet at a time and compares only the first 5 bytes against its command table.* There's no ACK/NAK (an unknown command produces total silence, indistinguishable from a valid silent command like `rotat`), no cross-packet accumulation (send `getva` as five 1-byte writes and you get nothing, even at zero delay), and matching is exact and case-sensitive. Baud rate is a no-op because it's USB CDC, not a real UART; DTR/RTS and BREAK are ignored.

**Polling faster makes it *worse*.** This one is genuinely counterintuitive. The measurement engine only refreshes at ~2–3 Hz, and every `getva` reply costs the single-core chip ~60 ms of AES + streaming that *starves its own measurement task*. So the harder you poll, the fewer distinct values you get back. Distinct-data capture actually peaks at a poll interval of about **0.22 s** and degrades on either side. My driver ends up flooring the interval at 0.2 s and offering a `--dedupe` flag. (This "faster is slower" coupling became a design target I got to *delete* later, in the custom firmware.)

**The datalogger is plaintext.** `gtrec` dumps the meter's rolling ~1 Hz log as 8-byte records — and unlike `getva`, they're *not* encrypted, just little-endian voltage and current, terminated by the ASCII `reck`. There's no command to start, stop, or clear it; logging is automatic whenever there's voltage on the port.

**The BLE radio is just weak.** Our unit advertises under the name of its Bluetooth module (`BT24-M`), not "TC66C", so you match on the service UUID `ffe0`, not the name. And the antenna is genuinely feeble — about −76 dBm from inches away. Connecting from a PC is a coin-flip unless the meter is right next to the antenna. That's hardware, not software, as I'd confirm much later.

---

## Act II — Getting inside: the key I couldn't compute, and the one I stole

To do anything real I needed the firmware, and the firmware ships **AES-256-ECB encrypted**, decrypted on-device by the bootloader as it flashes. This is real encryption, not obfuscation: the images have ~7.85 bits/byte of entropy, a single fixed block repeats ~465× (the ECB fingerprint of a constant plaintext block under one static key), and consecutive firmware versions differ in ~15.95 of every 16 bytes — full avalanche, no weak custom cipher. The key isn't in any host software (the PC app streams the `.bin` to the device verbatim). It lives only on the chip.

The chip is a **GigaDevice GD32F150G8** — an ARM Cortex-M3, 64 KB flash, 8 KB SRAM. So I had two independent ways at the key: pull it out of the chip's power consumption while it decrypts (a side-channel attack), or read it off the chip directly over the debug port. I went at both in parallel. One of them worked. It wasn't the one I spent five days on.

### The side-channel campaign (a five-day dead end)

The idea behind Correlation Power Analysis (CPA) is beautiful: when the chip decrypts a block, the tiny data-dependent variations in its power draw leak the intermediate values, and from enough traces you can statistically recover the key one byte at a time. I fully control the ciphertext (I'm the one streaming chunks to the bootloader), the decrypt happens a deterministic ~9.1 ms after each chunk, so it's a textbook setup.

I threw a lot of hardware at it, in escalating desperation:

- A **ChipWhisperer-Nano** sampling the chip's supply shunt. My first captures used the *wrong time window* — I triggered on the chunk arriving and captured the first 5 ms, completely missing the decrypt burst that lands ~9.9 ms later. Averaging eight full-length traces finally revealed the data-dependent activity at ~9–9.9 ms, right before the `OK` ack, and I moved the window there.
- A hand-wound **EM probe** over the die, Kapton-insulated, through a low-noise amplifier — I got it to a signal-to-noise ratio around 1.1, which should be plenty.
- A **Hantek 6022BE**, then finally an **OWON VDS1022I** at 100 MS/s with dual LNA front ends on both the supply shunt and the EM probe.

I learned real things. The AES leak on the supply rail is broadband but *low-frequency* — the board's decoupling capacitors low-pass the switching current — so an LNA with a 10 MHz high-pass corner puts the signal out of band and you see nothing; a 100 kHz LNA restores it. I confirmed a 16.9 MHz clock line was a real fundamental (not an alias) with sidebands that grow during the decrypt, and recovering its phase to resample onto a clock-synchronous grid measurably de-jittered the traces.

And none of it converged. Not with hardware power models, not with learned linear regression, not with EM-minus-supply common-mode subtraction, not with chosen-ciphertext isolation. For a while I had a "lead" — a machine-learning attack that named byte 0 as `0x72` with a plausible p-value — until I did the honest thing and ran a **permutation null**: shuffle the labels and see if the "signal" survives. It did. The winner was noise that my 256-way search was overfitting. The killing blow was recapturing byte 0 fresh: it named a *different* value (`0xaa`), and `0x72` didn't even make the top six. A real fixed-key leak names the same byte every time. Mine didn't.

There was a measurable wall — at 25 MHz analog bandwidth the whole 16-byte block is handled in ~0.6 µs, so no single time sample isolates one key byte's contribution; every key guess correlates about equally. But here's the part that stings, and I only learned it *after* I had the key by other means: **the entire attack was built on two wrong assumptions.** I'd assumed AES-**128** (it's AES-256, which needs the last *two* round keys to invert the schedule, not one), and my ground-truth oracle checked that the fill block decrypts to `0xff`×16 when it actually decrypts to `0x00`×16 — so even a *perfect* byte recovery would have been the wrong quantity, checked against an oracle that could never pass. Five days of careful signal processing, aimed at the wrong target with a broken verifier. The write-up in `cpa/RESULTS.md` is more honest with me than I was with myself at the time.

### The readout-protection wall

The other route was to read flash directly over SWD (the ARM debug port). To even attach a debugger I first had to find the pads — which is where the physical teardown began (more on that in Act IV), but the short version is I soldered wires to the 5-pad header and NRST pin, ran them to a Raspberry Pi Pico running a bit-banged SWD engine, and started probing.

The chip is locked at **RDP level 2** — the strongest GigaDevice readout protection — and I confirmed it empirically. I could catch the debug port alive in the power-on window and read the CPUID (that's how I confirmed it's a Cortex-M3, correcting an earlier guess of M0). But:

- Direct flash reads over the debug port **fault**.
- Using the on-chip DMA as a bus-master to copy flash somewhere readable — the published trick that broke a sibling chip — returned **all zeros** here (possibly a patched silicon revision).
- Loading a tiny stub into SRAM to read flash and copy it out **faults on the first read and locks up** — RDP2 blocks even the core's own data-reads of flash; only instruction-fetch works.
- And a nasty gotcha that bit me repeatedly and later went into the project notes: **attaching SWD under RDP2 halts the core.** A "frozen" SRAM value under the debugger does *not* mean your firmware crashed — it means the debugger stopped it. You verify liveness over serial with *nothing* attached.

Every direct path to flash was closed. The chip's readout protection was doing exactly its job.

### The heist

The breakthrough was a reframe. RDP2 gates *flash data reads*. It does **not** gate SRAM. And SWD lets you read SRAM in the background *while the firmware runs*, non-intrusively, without halting anything.

And where does the plaintext firmware — and the AES key schedule — necessarily exist in the clear? **In SRAM, during the bootloader's decrypt.** I didn't need to defeat flash protection at all. I needed to catch the bootloader in the act.

So (no power-cycle — that would wedge the USB of a half-updated app; I attach to the already-running target):

1. Boot into the bootloader (hold K1 while plugging in).
2. Send `update`, get `uprdy`.
3. Stream **one** encrypted 64-byte block from a real firmware image, get `OK`.
4. Attach and dump 8 KB of SRAM.

The post-decrypt dump had ~294 words of new nonzero state versus the idle bootloader — the decrypted block plus the AES key material. Diffing against a clean idle dump isolated it. At offset 0x2c8 sat a 32-byte value:

```
381255645a6b7aaabc6d5877f5c1fbff9805d1ac3c8335661a610c7ff73a638c
```

It decrypts every firmware version I have to a valid Cortex-M3 image — stack pointer in `0x2000xxxx`, reset vector in `0x0800xxxx` with the Thumb bit set. **That's the image key.** And of course, only *now* did I learn it was 256-bit and that the fill block decrypts to `0x00`×16 — the two facts that, had I known them a week earlier, would have told me my CPA attack was doomed from the first trace.

The lesson I keep from this: a clean architectural read of the target ("the plaintext must pass through SRAM, and SRAM isn't protected") beat a week of increasingly sophisticated signal processing aimed slightly wrong. Sometimes the move is to stop attacking the hard thing and notice the soft thing next to it.

---

## Act III — Reading the firmware in Ghidra

With the key, I could decrypt all four firmware versions and open them in Ghidra. This is where the project turned from "attack" into "understand," and there's one discovery that everything hinged on.

### The load address that makes or breaks everything

**The app is linked to run at `0x08002c00`, not `0x08000000`.** It sits immediately after the 11 KB bootloader. If you load the raw decrypted image at `0x08000000` — the obvious thing — the Thumb *code* still disassembles fine (it's mostly PC-relative), so it looks like it's working. But every absolute pointer lands 0x2c00 too low: the vector table, string tables, the AES-key pointer, the jump tables. Almost no cross-references form, and the analysis is quietly, thoroughly wrong. Rebase at `0x08002c00` and the whole thing snaps into focus. I confirmed the base three independent ways (the app's own reset vector, the bootloader's jump target, and the first instruction of `main` writing `0x08002c00` to the vector-table-offset register). The bootloader, separately, loads correctly at `0x08000000`.

I drove this with a mix of the Ghidra GUI (bridged to a live model via GhidraMCP on port 8080) and headless PyGhidra scripts for reproducibility.

### What the app is

The command dispatcher (`usb_command_dispatcher`) is a first-byte compare chain — which is exactly *why* string searches never found the commands: there are no verb strings and no immediate compares to grep for, just `if (buf[0] == 'g') …`. It confirmed the seven-command set from the firmware side, and confirmed the negative result too: **there is no PD-trigger verb in either the USB or BLE dispatcher.** The fast-charge engine exists in the binary, but nothing in the command path reaches it.

I found three AES-256 keys sitting back-to-back at `0x080098c3`: two "UID signature" keys and the `getva` telemetry key (the rdserialtool key, confirming the online value). And I found the thing that makes this chip dangerous to modify carelessly: an **anti-clone check**. Before its main loop, the app AES-encrypts the chip's factory UID and compares it against a stored signature; on mismatch, it **bricks into an infinite loop**. It only runs on a chip whose UID has been factory-signed. The saving grace, which I verified: a *modified app on the same chip* still passes, because the signature lives in a config page the app-flash never touches. You only brick if you try to relocate the app or wipe that page.

### What the bootloader is

The bootloader (v1.06) is small and, from a security standpoint, remarkably trusting. It reads the K1 button on pin PF0 at power-on: released → validate the app and jump to it; held → enter serial update mode. It exposes six verbs (not the two the PC app uses — there are also `uplogo`, `recali`, `writsn`, and a `gtsub` that leaks the chip UID). The update flow erases the 52 KB app region and writes AES-decrypted chunks at a monotonically increasing offset from `0x08002c00`.

The consequences of that design are the keys to everything I did next:

- **No integrity check.** No CRC, no magic number, no version check, no signature on the image. The `CHECK FIRMWARE...` on screen is just a status string. The image is written blind.
- **It physically cannot overwrite itself.** The write offset only ever increases from the fixed app base, so the bootloader, the UID signature, and the recovery path are all unreachable through the update protocol. **This is why recovery always works** — a bad flash never touches the bootloader, so holding K1 and reflashing a stock image always un-bricks it.
- Two keys sit adjacent in bootloader flash: the **image-decrypt key at `0x08002a8c`** (matching my SRAM-recovered value exactly) and a **UID-signature key at `0x08002a6c`**. I reproduced the per-device signature in Python from the chip UID and that key, byte for byte.

The net security posture: anyone who can hold a button and knows the (now-recovered) image key can flash arbitrary firmware onto this device, provided it's a valid ≤52 KB Cortex-M image and runs on the signed chip. Which, conveniently, described me.

---

## Act IV — The physical board (a parallel thread)

I've been referring to the teardown; let me back up, because it ran alongside the firmware work (roughly Aug 19–22) and it's a satisfying little sub-project of its own.

I needed two things from the physical board: the debug pads (to attach SWD for the key heist) and the full pin map (to eventually write firmware that drives the right GPIOs). Getting them meant identifying every component and reconstructing the netlist of a four-switch, densely-routed two-sided board about 22.6 × 40.2 mm.

**Component inventory.** Reading package markings off microscope frames and confirming them by live bus-probing later: the **GD32F150G8** MCU (QFN-28), a **DX-BT24-M** BLE module (the source of that `BT24-M` advertising name), a **BL24C256** 32 KB I²C EEPROM, a **TI INA226** current/voltage monitor, a **15 mΩ R015** sense shunt as the entire main current path, an **M5633B** 3.3 V LDO (an early misread called it a power-path switch — it isn't; there is *no* controllable load switch on the board, which is why VBUS can't be toggled in firmware), a **FUSB302** USB-PD PHY (invisible in the photos, found only when the I²C bus scan ACKed address 0x22 with device-ID 0x81), and a 0.96" **ST7735S** TFT.

**The stitching problem.** My primary imagery was a freehand microscope video pan, snapshotted into ~34 frames at wildly varying magnification. Automatic stitchers (OpenCV, Hugin) *fail badly* here, because a flat plane covered in repeated silkscreen produces high-confidence *false* matches — RANSAC happily locks onto a look-alike region a centimetre away with hundreds of inliers. One frame matched with 439 inliers and was wrong by a full centimetre; another doubled the "2020-11-04" date stamp. So I built a little drag-to-place tool: a browser canvas where I seed each frame's rough position by hand, and a CV backend then refines each seam with SIFT, but *rejects* any match that wants to move a frame farther than half a frame-width — which is exactly what stops the repeated-silk false locks. The recurring lesson, which I wrote at the top of the notes: **inlier count is not evidence** on a repetitive board; every placement needs an independent photometric or visual check.

**The upgrade that made it rigorous.** Eventually I depopulated the board with hot air and scanned the bare PCB on a flatbed at 2400 dpi. Evenly lit, perfectly flat, distortion-free. I registered the two sides to each other to **~2.4 µm** RMS using the mounting holes as fiducials — good enough to follow any via straight through the board — and traced the whole thing into a to-scale KiCad project on top of the scans. And the beauty of a *depopulated* board is that ambiguous routing stops being a guess: with no parts in the way, you just beep it out with a multimeter, and continuity means "same copper net," no exceptions.

The output of all this — the pin map — is what fed the firmware:

| Function | Pins |
|---|---|
| USB D+ soft-connect | PA9 (also the backlight enable — this dual use bit me later) |
| I²C0 (INA226 0x4A, EEPROM 0x50, FUSB302 0x22) | SCL=PB6, SDA=PB7 |
| On-chip ADC | ch0=PA0 NTC, ch1=PA1 D+, ch9=PB1 D− |
| TFT (ST7735S over SPI0) | SCK=PA5, MOSI=PA7, DC=PA4, RST=PA3; backlight PWM on PB3 |
| Buttons | K1=PF0 (bootloader), K2=PF1 (factory reset) |
| Charge trigger (D+/D− drive) | D+ via PA13/PA14, D− via PB4/PB5 |
| BLE module | USART1: TX=PA2, RX=PB0 |

I also mined the stock firmware for the *behaviour* behind those pins: the INA226 is used as a raw ΔΣ ADC (config `0x41A7`, no on-chip calibration) with all averaging and calibration done in software from per-unit constants in the EEPROM; the NTC is a 10 kΩ / β=3450 thermistor on the bottom of an 11.5 kΩ divider; the EEPROM's 32 KB is fully mapped (a wear-levelled energy ring at 0x0000, calibration at 0x2000/0x2200, the datalogger ring at 0x2400); and the display init sequence, gamma tables, and memory offsets were all recoverable verbatim. This is the reference that made a clean-room firmware *possible* rather than a guessing game.

---

## Act V — Bending the stock firmware

Before writing anything from scratch, I built a toolchain to patch the vendor firmware, as a lower-risk way to prove I understood it.

**`fwtool.py`** decrypts, patches, re-encrypts, and — the important part — *verifies*. It proves the cipher is bit-exact by checking that `encrypt(decrypt(vendor.bin)) == vendor.bin` for every firmware version. Patches are declared as `(offset, expected_old_bytes, new_bytes)` tuples so the tool refuses to apply a patch if the bytes aren't what it expects (wrong image, wrong offset). And because ECB blocks are independent, a build self-check asserts that a two-byte plaintext change alters *exactly* the corresponding two cipher blocks and nothing else. I proved the whole flash→observe→recover loop end-to-end first with a trivial "smoke" patch that flips the version string `V1.18`→`V1.19` on screen.

Then two real patches:

**Killing the "faster is slower" coupling.** Remember that pathology from Act I? I found its cause in the disassembly: everything runs in one cooperative loop, and `getva` sends its reply as 3×64 bytes with a `delay_ms(10)` busy-wait between blocks — ~20 ms per poll that *steals loop iterations from the sampler*. The first patch just trimmed the delays (10 ms → 1 ms). The better successor, `getva-txpoll`, replaces the fixed delay with a bounded spin on the USB endpoint's transmit-status flag — it waits exactly until the host has drained each block and no longer, and can never hang (a capped iteration count means worst case it behaves like a short delay). On hardware: 40/40 clean decodes, ~99% full replies under a back-to-back hammer.

I also investigated a deeper fix — moving the sampler into a timer interrupt to fully decouple measurement from polling — and **decided against it**, which I think is worth mentioning because knowing when to stop is part of the craft. The only spare-enough timer already runs the 1 Hz energy integration (change its rate and you break the energy math and the day clock); the samplers aren't re-entrant and get called from nine places inside the charge-protocol state machine; running a full bit-banged I²C read in an ISR risks blowing the 1 ms USB service budget; and a fault in interrupt context is an unrecoverable lockup with no equivalent of the bounded-spin safety net. The `getva-txpoll` patch already delivered "poll as fast as the ADC allows," and the ADC's ~2–3 Hz refresh is a hard physical ceiling regardless. So I left it.

**Reflashing with no button.** The most fun stock-firmware patch: a new `updat` command that lets the *running app* reflash itself over USB with no bootloader dance. This is tricky because the app runs *from* the region it needs to erase, so the flasher has to run from SRAM and stream, and once the app flash is gone there's no USB driver left — so the SRAM stub polls the USB peripheral registers directly and reuses the *bootloader's* flash routines by absolute address (the bootloader is never erased). I injected it as a small flash-resident hook into a verified-empty hole in the image, with a branch planted over the dispatcher's prologue. The chunks are sent as *plaintext* (the on-device image is decrypted anyway, so this avoids baking an AES inverse into the stub). It self-updates in about a second, ~9× faster than the bootloader path, and it's self-hosting — the `updat` path survives updating itself. Recovery, as always, stays K1 + reflash.

At this point I could measure, patch, and reflash the meter at will. But patching someone else's firmware only goes so far. I wanted the whole thing.

---

## Act VI — A new brain: the from-scratch firmware

The custom firmware is written in Rust (`no_std`), flashes into the 52 KB app slot, and keeps the stock bootloader so recovery always works. It's wire-compatible — it enumerates as the same USB device and answers the stock commands — but everything underneath is mine and fully understood. I labelled milestones M0 through M4. The core of them landed over three days; the persistence, the extra screens, and the Bluetooth link were finished in later sessions.

### M0 — self-hosting

The first goal was simply to boot my own code, enumerate as the stock CDC device, answer `query`, and reflash itself via `updat`. I wrote a C reference first to validate the boot contract on hardware, then ported it to Rust line-for-line. Once a Rust image could flash the *next* Rust image over USB, the C reference was retired and the iterate loop became `make flash` — no button pressing, ever. That self-hosting property is what the project is named around.

The boot contract has four hard-won rules, each of which cost real debugging:

1. The initial stack pointer must be `0x20000E40`, *not* top-of-RAM — the exact top faults the bootloader's hand-off, for reasons I never fully pinned down. `0x20000E40` is proven-good, so that's what ships.
2. The vector-table offset register must be set to `0x08002c00` early — the bootloader jumps to the app without setting it.
3. **PA9 is the USB D+ soft-connect.** The bootloader hands off with the bus still connected and the host bound to the *bootloader*, so the app must physically toggle D+ (drive PA9 low, bring USB up, PA9 high) or the host never re-enumerates it. The device otherwise inits perfectly and looks dead.
4. A complete vector table and a cleared NVIC, or an unhandled interrupt branches into random code and locks up.

I moved to the Rust embedded ecosystem USB stack (`usb-device` + `usbd-serial` + `stm32-usbd`, since the GD32's USB peripheral is the STM32F0/F1-style IP) and wrote a pure-Rust SRAM flasher. That flasher is a masterclass in constraints, because it erases the very flash it was loaded from: raw pointers only, no operation that might emit a hidden `memclr` call into flash, its own inline assembly for the barriers, reply strings materialised as integer literals, and the bootloader's flash functions called through absolute-address function pointers. A build-time gate disassembles the flasher and *fails the build* if any instruction references app flash.

That gate earned its keep immediately, via one of my favourite bugs. I'd placed the flasher's linker section with `INSERT AFTER .data`. The Rust runtime deliberately extends its startup memory-copy around sections after `.data`, *assuming a contiguous address* — but my flasher's run address is elsewhere, so the runtime **smeared ~6 KB of flash across all of low RAM on every boot**. It was completely harmless while there were zero static variables — and then instantly fatal the day I added the peripheral-access crate, whose one flag static got clobbered into a panic. The SRAM debug markers pointed straight at the line. Fix: place it after `.uninit`, pin the RAM boundaries, and add *another* build gate asserting the memory sections all land in real RAM.

### M1 — real, calibrated measurement

I brought up each peripheral behind a host-triggered "dev verb" first — nothing runs at boot, every wait is bounded, faults return staged diagnostics — so a mistake is always recoverable by power-cycle and never bricks the boot path. Only once proven does a feature graduate to the always-on path.

The I²C bus scan found the INA226 (confirmed genuine via its manufacturer-ID `0x5449` = "TI" and die-ID `0x2260`) and the EEPROM. I dumped the whole 32 KB EEPROM twice (identical hashes) before ever contemplating a write, as a permanent calibration backup. Then I implemented the calibration model recovered from the stock firmware — integer, multiply-before-divide, with the per-unit gain/offset/zero-reference constants read from the EEPROM — and reproduced a known voltage (5.2084 V) *exactly* against the raw registers. The `getva` builder embeds a known-good decrypted template, patches the live fields at their offsets, recomputes the three per-block CRC16 checksums, and AES-encrypts. Existing host tools couldn't tell it from a real meter.

The on-chip ADC (for temperature and D±) had a great gotcha: its clock comes from a dedicated 14 MHz oscillator that's *off at reset*, and the ADC's self-calibration hangs forever without it. The staged dev verb caught this cleanly (it returned "stage 4: oscillator never stabilised"). My first pass at the ADC channel map was empirical, and partly wrong: the stock code reads its three analog inputs as "inserted-group ranks" 1/2/3, and I'd mistaken those rank indices for channel numbers. Going back to the stock ADC init settled it — **ch0 (PA0) is the NTC, ch1 (PA1) is D+, ch9 (PB1) is D−** — and the pin I'd been reading as D+ (PB0) turned out to be the Bluetooth module's UART output, idling at 3.3 V. One board design, one channel map. My first temperature model (a fitted one) reported an impossible 11.6 °C on a cool board; adopting the stock β-model gave a sane 27 °C, and it now runs in integer fixed point, proven bit-identical to the stock floating-point formula over every possible input by an exhaustive host-side test. Details matter.

### M1.5 / M2 / M3 — streaming, a display, persistence, and an architecture

A dense middle stretch:

- A **hardware-paced sampler**: a timer interrupt reads the INA226, so measurements no longer depend on how fast (or whether) a host polls. It started at 250 Hz; it now ticks every 2.24 ms and reads both the shunt and bus voltage on each tick, which catches about **98% of the conversions the INA226 actually makes**. The averaging window and the screen refresh rate are settings, and the energy counters integrate every sample.

- A plaintext **`rawva`** streaming verb with a window sequence number, because a full encrypted `getva` costs ~20 ms of software AES — the entire reply budget — while `rawva` is microseconds. It streams at 700 polls/second with zero missed sampler windows.
- The device became a **composite USB device that emulates a ChargerLAB POWER-Z KM003C** on a vendor interface, so the mainline Linux `powerz` hwmon driver binds it and `sensors` just works. This is actually the *priority* protocol now; the stock TC66 CDC commands ride along for compatibility. It goes well past what `powerz` needs: the KM003C's **graph mode** streams at 2, 10, 50, or **1000 samples per second**, and the open-source [km003c-rs](https://github.com/okhsunrog/km003c-rs) CLI and GUI work against it unmodified — live V/I plots with the PD traffic in the same timeline.
- The **ST7735S display**, driven by a non-blocking DMA pixel engine (16-bit SPI frames on a DMA channel, pumped from the main loop so it never blocks USB or the sampler), with a 1-bpp font generated from Inconsolata Bold and a three-line V/I/P screen. The first frame draws *before* USB enumeration, so the screen is live from power-on. An overnight incident — a static screen caused real LCD image persistence — drove a 15-second screen-off timeout, with the wake condition later refined from "no input" to "no *change* in input" (a steady load parked on the port is as static as no load).
- **Six screens**, stepped with K1: V/A/W, energy and run time, temperature and D±, a full-screen scrolling V/I plot, a USB-PD info screen, and a charger-detection screen (both in Act VII). The plot uses the ST7735S's *hardware* vertical scroll, so each new sample costs one column of pixels rather than a redraw. Holding K2 resets the energy counters.
- **Persistence (M2)** in the stock firmware's own EEPROM layout: settings, the wear-levelled energy ring, and the datalogger that `gtrec` reads, so switching between stock and custom firmware keeps them. All writes go through one driver that verifies every page and hard-refuses the factory calibration words.
- A hardware **watchdog** (~13 s), fed once per main-loop pass and inside the self-flasher.

And underneath it all, the architecture had to change from "poll everything in one loop" to interrupt-driven, which is where the chip fought back hardest.

### QUIRKS.md — where the silicon misbehaves

I keep a file called `QUIRKS.md` of measured, reproducible GD32F150 misbehaviours, using the *stock firmware* (production code on this exact silicon) as the discriminator for "is this the chip or is this me." The greatest hits:

- **NAK traffic stalls the CPU's own peripheral accesses by 1–5 ms each.** While the USB host is NAK-polling any endpoint, reads of unrelated peripheral registers wait-state for *milliseconds*. I proved it by bracketing ~300 instructions with timer timestamps and measuring 1.1 ms — while a register-only busy loop in the same function ran at normal speed. The polled main loop collapsed to ~70 iterations/second under a NAK storm, self-sustaining because a slow loop keeps the endpoint unserviced. On genuine STM32 silicon the hardware handles NAKs for free. The fix was structural: pre-armed replies (no steady-state NAK storms), a core-internal timebase that can't stall, and moving USB service into the interrupt.
- **USB completion events get lost, and reading the status register at ISR entry actually *consumes* events.** An A/B test was unambiguous: with the status-register read present, 0 of 401 replies produced a completion callback; delete that read and it's 50%. The chip's manual, unlike ST's, never promises the level-triggered guarantee I was relying on. The stock firmware's own answer — a drain loop that re-checks per-endpoint flags rather than the global status register — is what I ended up mirroring.
- **The GD32F150 only fetches zero-wait from the first 32 KB of flash;** above that, every instruction fetch is ~10× slower. My linker had put the entire AES implementation above the boundary. A linker script (`fast.x`) that pins the hot path — interrupts, USB, AES, main loop — below 32 KB took `getva` from **48.6 ms to 2.3 ms**, and incidentally explained away most of the confounds in the two quirks above. To find *what* to pin, I built a poor-man's PC profiler: the chip's debug cycle counter is a non-functional stub, so I sample the interrupted program counter off the exception stack frame on every sampler tick into an SRAM ring and dump it over serial.

There's also a permanent debug reality behind all of it, imposed by RDP2: SWD `halt`/`reset`/`step` lock the core, and even a background memory read can leave it halted, so `probe-rs`/RTT/`defmt` are all unusable. My entire debug channel is the SRAM marker map plus the dev verbs, and liveness is confirmed over serial with no debugger attached. Debugging embedded code with essentially `printf`-to-SRAM and a multimeter is a throwback, but it works.

### The stack-gate

One more safety mechanism, because it embodies the central fear of this whole project. A stack overflow *on the boot path* hardfaults **before** the D+ pull-up — producing an invisible dead board that needs physical K1 recovery. That shipped once, as build 35, and it's a miserable way to find out. So every build runs a `stack_gate.py` that computes a conservative worst-case stack depth from the disassembled call graph — discovering every interrupt handler from the vector table, charging the deepest handler chain, treating recursion as a hard error — and checks it against the RAM budget. When the budget got tight, the release valve was a `.hibss` section that parks large statics (the AES key schedule, a PD-capture ring) in the 1 KB of RAM above the stack fence, which raises the budget byte-for-byte. Switching to size-optimised builds with fat LTO later dropped the flash from ~49.5 KB to ~35.6 KB and gave real headroom back — which the later features have since used up. The flash is full again (about 150 bytes spare in the release build); the last big savings came from replacing the AES crate with a compact hand-written AES-256 (−4.3 KB, and `getva` got three times faster) and dropping linker thunks and floating-point math. The rule for the next feature: room gets made on purpose, never by quietly changing code that already works.

---

## Act VII — The crown jewels: making it do the thing I came for

Everything so far was infrastructure. This is the payoff — exposing over USB the fast-charge triggers the stock unit keeps locked behind its buttons. The triggers landed in one intense day (the M4 milestone), with the PD sniffer and charger detection following in October, all building on the FUSB302 PD PHY, the D+/D− drive GPIOs, and the pin map from the teardown. Every one of these was verified against real chargers on the bench.

### USB-PD sink and PPS

The FUSB302 (I²C address 0x22) does the Type-C physical layer — the line coding, the CRC, the automatic GoodCRC reply — while my `pd.rs` implements the protocol/policy half of a minimal PD 2.0/3.0 sink: detect attach and cable orientation, bring up the BMC link on the active CC pin, collect the source's capabilities, and negotiate a contract. Four host verbs (`pdctl`/`pdsta`/`pdcap`/`pdreq`) arm it, report state, list the source's power profiles, and request one.

It worked, eventually, against an Apple 96 W charger at 5/9/15/20 V — after two instructive bench bugs. First, a source you plug in *before* arming has already given up trying to talk PD (it exhausted its capability-message retries against the chip's silent default), so `arm()` now drops the pulldowns for ~600 ms to force the source to see a fresh attach. Second, a timeout was comparing against a stale timestamp captured *before* several milliseconds of I²C, so the subtraction wrapped negative and fired the `Accept` timeout instantly — discarding a perfectly good exchange. The instrumentation I added to catch it (logging the last few received message headers) is what found it.

PD 3.0 and PPS (the fine-grained adjustable voltage mode) came next, with a rule learned on the bench: lead with revision 2.0 and only upgrade to 3.0 after the source declares 3.0 in its capabilities, because a 3.0-stamped reply to a 2.0 message made the Apple charger reset the link. PPS needs a keepalive (re-request within ~10 s or the contract dies), so the engine re-sends the standing request every 8 s. Verified: PPS fine-stepping across 3.3 / 5 / 9 / 11 V, each tracked within ~50 mV on the meter's own sampler, and a 9 V PPS contract held steady for 24 seconds — past the deadline, confirming the keepalive.

I also exposed the CC-pin voltages (via a binary search on the FUSB302's built-in comparator) and re-exported every PD message the meter sends or receives — in the real KM003C wire format — over the POWER-Z vendor endpoint, so a KM003C protocol tool can watch the meter's own live PD traffic. That last feature was gated not by protocol difficulty but by 336 bytes of RAM, which is what forced the `.hibss` high-RAM statics into existence.

### Listening in: a PD sniffer

The FUSB302 can also listen without taking part. In sniffer mode the meter stops presenting itself as a sink and records whatever a charger and a device say to each other *through* it: source capabilities, requests, vendor messages, the **cable's e-marker** answering on SOP′/SOP″, attaches and detaches, hard resets, and even packets that fail their CRC. It works in either plug orientation and can make the pair renegotiate without replugging. The capture goes out in the same KM003C PD format, and a host tool (`pdcap.py`) decodes every message type field by field, using layouts taken from the USB PD R3.2 spec. The PD info screen shows the same thing on the meter itself.

Two bench lessons. First, the FUSB302's BMC slicer threshold matters: at the default, one VDM in every capture went missing, and nudging the threshold brought it back. Second, *my unit has a board fault*: the OUT port's CC2 contact is nearly open, so sink attach and hard reset only work with the line on CC1. Both CC nodes are unloaded at DC and no GPIO or ADC pin touches CC, so there's nothing firmware can do about it.

The sink side learned the same trick. At attach it listens to the source probing the cable's e-marker, and it can ask a PD 3.x source for its extended capabilities, status, and identity.

### QuickCharge 2.0 and 3.0

QuickCharge is a different animal — not a CC/PD protocol at all, but fixed DC voltages presented on the pass-through D+/D− *data* lines. So it's a separate, GPIO-only engine driving the resistor network on those four pins (recovered pin-for-pin from the stock firmware). QC2.0's discrete levels verified at 5.11 / 9.13 / 12.14 V against a Class-A charger. QC3.0's continuous mode — where you pulse a line to step ±200 mV — verified bidirectionally: base 5.31 V, +10 steps to exactly 7.30 V (200 mV × 10), and back. Two gotchas: back-to-back pulses merge into one edge unless you leave a ~20 ms dwell between them, and my initial down-pulse had two pins swapped so it did nothing. (A standing hazard: the D+ drive pins are *also* SWDIO/SWCLK, so SWD is dead while a QC trigger is active.)

### Samsung AFC — the debugging saga

This is the richest debugging arc in the project, and the honest version is better than the tidy one.

Samsung's Adaptive Fast Charging is a single-wire serial protocol on D−. I implemented it straight from my reverse-engineering notes as **152 µs NRZ bits, MSB-first, odd parity, with a start pattern** — and it didn't trigger the charger. Six iterations followed:

1. Added sync pings — no change, but the real blocker turned out to be *diagnostic*: my oscilloscope was showing a flat line during a deliberate drive, because the probe wasn't referenced to the driven node. My own firmware's ADC confirmed the pin *was* being driven. No scope visibility means you're debugging a microsecond-timed protocol blind.
2. With the scope finally on the right node, I found the actual code bug: my bit delays used a busy-loop, but that code lived in *slow flash* (above the 32 KB fast boundary), so each 152 µs bit was actually taking ~1090 µs. Switching to the free-running SysTick counter (real time, independent of flash fetch speed) fixed the timing, and the scope now decoded my output as byte-perfect — exactly the right bits, parity, and framing. **And the charger still didn't switch.**
3. So I went looking outside my byte framing, added an open-drain acknowledgement bracket around each byte — still nothing.
4. Then a fresh scope capture of the *stock* TC66's output convinced me the encoding was **pulse-width, not NRZ** — and I set out to rebuild the whole engine. This felt like the breakthrough. **It was a wrong turn.** A better, higher-sample-rate capture reversed it: the encoding *is* NRZ, exactly as I'd had it since step 2; the "pulse-width" reading was an artifact of undersampling and the earlier probe problems.

The thing that actually made it work was never the encoding at all. It was three things: the protocol is **closed-loop** (the master drives a "ping," then *reads the pin back* to follow the charger's response ping before each byte — an open-loop replay engages the charger but never completes the handshake); the entry sequence releases the lines to high-impedance and lets the charger *source* the D− level rather than driving it; and the bit timing has to run with interrupts masked so the sampler interrupt doesn't stretch a bit cell mid-byte. With all three, `afctr` drives a real Samsung charger from 5 V to a measured 8.92 V, reliably across power-cycles. The D+/D− drive hardware had been provably correct the entire time (QuickCharge used the same pins) — which is *exactly* why the encoding looked like the culprit for so long when the real gap was the handshake protocol.

The lesson I took: when the low-level output is provably correct and it still doesn't work, stop re-examining the low level. The bug is in the protocol *state*, not the bits.

### Huawei FCP/SCP

Huawei's FCP and SCP ride the same physical layer as AFC, so they were scaffolding on the same engine — multi-byte register packets with a Huawei-specific CRC-8 (verified bit-exact against all my reverse-engineered examples) and three leading sync pings. But I have **no Huawei charger on the bench**, so these are explicitly *untested*, with three load-bearing assumptions marked in the code (most critically, whether the parity bit is per-byte or runs across the whole packet). Shipping untested code labelled as untested, with the specific unknowns called out, is the honest thing to do — but it's not "done."

### Charger detection, on the device

The detection screen answers "what does this charger support?" without ever raising the voltage: **it never goes above 5 V**. One K2 press reads the Type-C current advertisement, checks for Apple/Samsung D± dividers, probes QuickCharge and AFC, and lists every PD offer. It runs as a state machine stepped from the main loop, so USB stays responsive while it works, and it refuses to start if something is plugged into the other port, because a probe that accidentally raised the voltage must never reach a device.

### BLE

Finally, the BLE link: I reimplemented the stock transport for the on-board BT24 module over its UART, with DMA for both receive (a ring) and transmit, so it costs zero interrupts — I hold a strict two-interrupt budget: only USB and the sampler. I recovered from the stock firmware that the module is on USART1 with TX on PA2, RX on PB0 (the old notes said USART0 — wrong). The bench corrected the other half of my notes: the stock firmware never provisions this unit's module, so it runs at its power-on default of **9600 baud**. The module answers every AT *query* but rejects every persistent write with `ERROR=104`, so there's no changing it. The stock `b`-prefixed verbs are always on, so the vendor's Bluetooth protocol works as it did on stock.

---

## What it adds up to

Eleven days to get here, a few more bursts to finish it, one $30 meter, and it now does everything I originally wanted and a lot I didn't know I wanted:

- I understand its wire protocol completely, including the ways it's stranger than documented.
- I recovered its firmware encryption key — not by the sophisticated attack I spent five days on, but by noticing the plaintext had to pass through unprotected SRAM.
- I have a full Ghidra map of both the app and the bootloader, and a verified toolchain to patch and reflash them.
- I have a to-scale KiCad reconstruction of the PCB and a confirmed pin map.
- And I have a from-scratch Rust firmware that measures with a hardware-paced sampler (no more "faster is slower"), emulates a POWER-Z well enough for the mainline Linux driver and 1000-sample/s streaming tools, sniffs and decodes USB-PD conversations, identifies chargers on its own screen, talks Bluetooth, drives the display, self-flashes over USB with no button, and — the thing that started it all — **triggers USB-PD, PPS, QuickCharge 2.0/3.0, and Samsung AFC on command**, all bench-verified against real chargers.
- And it's public: the repo has CI that builds and runs every safety gate on each push, and each release is a single encrypted image in the bootloader's own format plus a standalone flashing script, `tc66flash.py`.

A few things I'll carry forward. First, the cheapest win in the whole project was a *reframe* (SRAM, not flash), and it beat a week of signal processing pointed slightly wrong — the CPA campaign was aimed at AES-128 with a broken oracle and could never have succeeded, which I only learned once I'd won by other means. Second, "the low-level output is provably correct and it still doesn't work" almost always means the bug is one layer *up* — the AFC handshake, not the AFC bits. Third, on a locked, hard-to-debug chip where a boot-path fault means physical recovery, the boring safety infrastructure — the build-time gates, the recovery path that can't be bricked, the dev-verb-first bring-up — isn't overhead. It's the only reason the fun parts were ever affordable to attempt.

Recovery, incidentally, still always works. The bootloader is never touched by any flash. Hold K1, reflash stock (`tc66flash.py stock` downloads RuiDeng's own V1.18 from their server, so I never redistribute it), and it's a boring, perfect little power meter again — until the next idea.

---

*Everything here was done on hardware I own, against my own devices. The firmware images, vendor apps, and datasheets aren't redistributed. The custom firmware, its host tools, and its documentation are at [github.com/FletcherD/tc66-custom-fw](https://github.com/FletcherD/tc66-custom-fw).*
