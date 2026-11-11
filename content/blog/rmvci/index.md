+++
title = "rMVCI: a Rust driver for the Mini-VCI cable, and how Techstream stores its vehicle data"
description = "Reverse-engineering the Mini-VCI J2534 cable's firmware, writing a Rust driver for it, decoding Toyota Techstream's diagnostic database, and reading my Prius's A/C servo position on Linux. Part one."
date = 2026-10-16

[taxonomies]
tags = ["reverse engineering", "Rust", "automotive", "firmware", "Ghidra"]

[extra]
toc = true
+++

**[rmvci on GitHub](https://github.com/FletcherD/rmvci)** • **[rmvci-core 0.1.0 release](https://github.com/FletcherD/rmvci/releases/tag/v0.1.0)**

Techstream is Toyota's dealer diagnostic software. It's a 32-bit Windows application, and it talks to the car through a J2534 pass-through adapter. The cheap clone adapter most people use is the Mini-VCI. It has one Linux driver, [libMVCI](https://github.com/aselafernando/libMVCI) by Asela Fernando, which supports K-line but not CAN.

I wanted to read one value from my Gen-2 (NHW20) Prius on Linux: the position of the air-mix blend door servo in the climate control, commanded versus actual. That took about a week in August 2026 and turned into a new driver and a decoder for Techstream's database. This is part one, because the project isn't finished.

## Looking for the request

[OBDb](https://github.com/OBDb) has per-vehicle JSON databases of diagnostic signals. Its Prius repo covers the powertrain, hybrid battery and chassis but not the A/C amplifier. The shared Toyota base repo has a climate block used by the Camry, RX and 4Runner: CAN ID `7C4`, KWP2000 read `21 43`, with byte 0 the target and byte 1 the actual servo position. I assumed that would work on the Prius. It doesn't, as I found out later.

To send anything on CAN I needed the cable to do CAN, so the first question was whether the limitation was in the cable or in libMVCI.

## The cable firmware

The cable is an FTDI FT232R feeding an NXP LPC2119 running firmware that identifies itself as "J2534 MINI V1.03". The Windows driver DLL is protected by a commercial packer, with a decoy import table and an encrypted code section.

The firmware was easier. A public firmware-restore image for this cable has code read protection off, and Ghidra found 480 functions in it: an ARM bootloader at `0x0` and the J2534 application at `0x8000`. Probing my cable confirmed that the firmware accepts protocol IDs 1 through 6, which include CAN and ISO 15765. So CAN works in the cable and was only missing from the driver.

Reading the firmware turned up several things a driver has to know about:

- The CAN acceptance filter comes up enabled and empty. A channel with no filter receives nothing, which looks like a dead bus.
- The filter mask isn't a mask. The firmware programs a range, `[pattern & mask, pattern | ~mask]`, so the usual `0x7FF` mask matches nothing useful and matching one CAN ID needs `FF FF FF FF`.
- There's one filter per ID width. Installing another silently replaces it.
- `SET_CONFIG` returns success for unknown parameters and discards them. The ISO-TP flow-control settings (block size, separation time, padding) are stored and never read.
- Protocol IDs 7–11 crash the MCU. The firmware indexes a six-entry table with `id - 1` and only checks for zero. Recovering needs a port reopen, which pulses DTR/RTS to reset the chip.
- The firmware's ISO-TP ignores the receiver's block size, uses a fixed 1 ms separation time, and truncates the first-frame length to 8 bits, so messages over 255 bytes break. This probably explains the "~48-byte limit" people report with these cables: 6 + 6×7 = 48 is an ECU advertising a block size of 6 and giving up when the cable overruns it.

I also found four errors in libMVCI's protocol notes. Lengths are 16-bit, not 8. The status byte is at offset 3, not the end. The trailing bytes are uninitialized stack. And its "disconnect" command is actually the one that tears down the encrypted session, which is why it has to re-handshake after disconnecting.

For bench testing I wired the cable to a CH340 USB-CAN analyzer and wrote a fake ECU in Python that answers `21 43`. The cable answered the fake ECU's multi-frame response with the flow-control frame `30 00 01`, which is the hard-coded value I'd seen in the disassembly.

## rMVCI

I first added CAN to libMVCI on a branch: CAN, ISO-TP, the protocol fixes, all 24 test vectors passing, no K-line regressions. But I wanted a different design, so on August 7 I wrote a new driver in Rust. (The libMVCI branch is still local. I need to upstream it.)

- A sans-IO codec, so the wire protocol can be tested without hardware. libMVCI's 24 captured test vectors are ported byte-for-byte.
- A transport trait with serial, direct-USB and mock backends, and one actor thread that owns the port.
- Typed channels, `Channel<P>`, where the protocol is a type. A filter of the wrong width or an out-of-range protocol ID can't be expressed.
- ISO-TP either through the firmware (fast, with the limits above) or host-side over raw CAN (one USB round trip per frame, but correct block size, separation time and long messages).
- A `cdylib` exporting the 14 standard J2534 `PassThru*` functions, so Windows tools like Techstream can load it.

Against a fake ECU asking for a block size of 2 and a 50 ms separation time, the firmware path sent 7 frames with about 0 ms gaps. The host path paused after 2, at 50.0 ms.

libMVCI pings the cable every 15 ms because "the adapter resets if left idle". I tested that by idling for 5, 60, 300 and 900 seconds and then checking the session, the channel, and a full request and response. All of them worked after every idle period, so rMVCI sends no keepalive. It detects a stuck adapter by counting unanswered real requests.

I also got it running on a rooted Android phone, which exposed four bugs in the JNI layer: USB permission always denied because of an immutable `PendingIntent`, the USB serial library rejecting reads smaller than 64 bytes, a SIGKILL from an uncleared Java exception on the actor thread, and a 5-second close caused by a cached ISO-TP object holding a second handle to the actor. On Linux I added a direct-USB backend using `nusb`, which can set the FTDI latency timer without root and cut exchange time from 48 ms to 30 ms.

## Techstream's database

With the cable working I still didn't know the right request for my car, so I looked at how Techstream knows.

I couldn't get Techstream running under Wine. Its data is in Microsoft Access databases behind ADO, which needed the Jet engine installed by hand, native ADO DLLs, and a larger stack reserve in the PE header. After all that, every version from v7 to v18 died with an uncaught C++ exception. I later traced it to Wine's 32-bit C++ exception handling: an exception thrown from inside a `catch` block loses the enclosing handler. So I used a Windows XP VM with the cable passed through over USB.

Techstream turns out to contain very little vehicle knowledge in code. It's a generic engine plus a database:

- `.vds` files are renamed Access databases with vehicle lists and ECU addressing.
- About 437 `.ddb` files, one per ECU family, hold every signal, trouble code, actuator test and customization setting, including the request bytes to send and the byte and bit slice to decode from the reply.
- Names are in a separate string table: 183,244 strings, UTF-16, LZSS-compressed.

So an open replacement only needs the database format reverse-engineered once, not each vehicle.

The `.ddb` families are protocol generations. P3 ECUs use KWP2000 `21 <LID>` reads, P5 ECUs use UDS `22 <DID>`, and P4 is a mix. Matching trouble codes identified `A_C_P3.ddb` as my car's A/C amplifier, which meant the `21 43` request from OBDb belonged to a newer generation.

Some details from getting the decoding right:

- String IDs were being masked to 16 bits, but ABS uses IDs around 95,000. The same bug appeared in three different fields. Fixing it properly took bad names from 6,124 to 9.
- Scaling is `(raw × num / den + offset) / 10^decimals`. The ambient temperature sensor came out as `×0.35 − 23.3 °C`, which matches OBDb.
- Checked against OBDb, A/C names matched 12 of 12 and the P5 generation about 99%.
- The vehicle-to-ECU map for all 2,480 vehicles comes from the database's menu tables.
- One record decides which bus an ECU is on: an address of `0xFF` or less means K-line, and larger means CAN.

By that rule, my car's A/C amplifier isn't on CAN at all.

## Capturing what Techstream sends

To see Techstream's requests I wrote a small logging DLL with the 14 `PassThru*` entry points that records every call and streams the log to my Linux host. (The first build crashed Techstream on launch because it opened a socket from `DllMain`, which deadlocks under the loader lock.)

When I selected my car and opened the Air Conditioner screen, Techstream never used CAN `7C4`. It switched to K-line, ISO 14230, address `0x98`, as the database rule predicted, sent a few identification reads, then polled `21 E1` repeatedly. With some OCR-driven automation of the VM I captured request frames for 11 of the car's 15 ECUs.

The logger was faking ECU responses at this stage, so it showed real requests but no real replies. I misread the capture because of that: I decided `21 E1` was a bulk "read the whole data list" command. On the real car it returns a two-byte trouble-code count, and Techstream uses the per-LID reads from the database for live data.

## On the car

On August 9 I plugged the cable into the car, connected to a Raspberry Pi running `rmvci-bridge`, a small TCP service that forwards the driver's transport primitives so I could work from my desk.

The first attempt failed. FAST_INIT returned a single status byte, `08`, and every write timed out. I had skipped the bring-up sequence Techstream does before using K-line: twelve `SET_CONFIG` timing parameters, plus a periodic-message clear before init and before each write. These turned out to be ordinary J2534 timing IDs (9600 baud, `P1_MAX`, `TINIL` and so on) with Toyota's values.

On the second attempt FAST_INIT returned the full key bytes, `83 f0 98 c1 e9 8f 44`, but every reply seemed to belong to the previous request. The A/C amplifier answers `7f 21 78` ("response pending") first and sends the real `61 <LID>` a moment later. Once the driver drained the pending frames:

| Signal | Request | Raw | Value |
|---|---|---|---|
| Air-mix damper, actual | `21 0D` | 9 | −9.5 % |
| Air-mix damper, target | `21 13` | 8 | −10.0 % |
| Ambient temperature | `21 02` | 164 | 34.1 °C |
| Blower | `21 1C` | 0 | off |

That evening I checked the extracted vehicle map against every ECU in the car using only reads:

- 13 of 13 ECUs answered at the bus and address the database predicted: eight on K-line (A/C, ABS, body, steering, gateway, ignition, immobiliser, transmission) and three on CAN (engine, hybrid, hybrid battery).
- The A/C values were plausible for a car parked in the sun: ambient 33 °C, cabin 39 °C, coolant 62.6 °C.
- The engine ECU reported a stored P0420. A 50-byte multi-frame ISO-TP response from the hybrid ECU reassembled correctly.
- Two LIDs on the A/C amp (`21 52` and the DTC read) produced an endless stream of "response pending" frames and left the amplifier stuck until re-initialised. The driver now drains these and re-inits. `21 52` turned out to be the one A/C LID the database flags as unsupported on this ECU.
- Five chassis ECUs answered their identification read but not live data.

## Techstream through rMVCI

Next I changed the logging DLL to forward each J2534 call to a server on my PC, which replayed it on rMVCI over TCP to the Pi, the cable and the car. Techstream in the VM connected to the car and showed real data: 686 ECU responses in that session, including 65 from the ABS and 51 from the body ECU, which my own tool hadn't been able to read.

The capture showed why. When a LID goes unanswered, Techstream re-runs FAST_INIT after two empty reads and carries on. My driver does that now. The capture also corrected my static analysis: the KWP trouble-code read is service `0x13`, not `0x18`.

A later capture, with me clicking through every ECU and data list, covered 15 K-line and 5 CAN ECUs and showed how the CAN-side data lists work: `A8 01` enumerates the PIDs, `A1` defines a poll set, and `A2` polls it. Active tests are service `30`.

Techstream v18 hung on connect after asking for 10400 baud on K-line, and I suspected the clone cable. I sniffed the genuine Windows driver talking to the cable, decrypted the session, and compared it to mine. They were byte-identical. Techstream tries 10400, gets refused, and falls back to 9600.

## Running the engine headless

Driving a Windows GUI in a VM is slow, so I tried running Techstream's diagnostic engine without the GUI. The engine is a chain of DLLs behind a `DiagAPI` facade, and the Wine crash is in code that chain doesn't call. I built a harness that loads the engine DLLs under Wine, with the J2534 layer answered by replaying my car captures.

After a day of finding undocumented preconditions, it read the VIN from the replay, identified "NHW20 1NZFXE", and produced request frames matching the real capture byte for byte: 98 of 98 for the A/C amplifier, plus data lists for six of the nine K-line ECUs. The harness enters the live-value poll path but fails a precondition check before sending any `21 <LID>`, and I haven't found the cause.

## The extractor

By August 10 my findings were spread over forty-odd markdown files that sometimes contradicted each other, so I spent a day consolidating them into one `DATA_MODEL.md` and marking the rest as current, log, or superseded.

The extraction scripts became a pipeline, `techstream_extract`. Run against a Techstream installation, it takes about four seconds and produces, for 415 ECUs and 2,480 vehicles: 56,684 data-list signals, 61,072 trouble codes, 5,995 active tests, 3,451 freeze-frame items, 2,139 customisation settings, and the vehicle-to-ECU map. The tool is published. Its output isn't, because it derives from Toyota's database.

The rewrite found bugs the one-off scripts had hidden. The field I'd called the "mask" is a validity mask, and using it to extract values would have truncated 23% of bit-sliced signals. Trouble codes aren't unique within an ECU. Analog active tests take two-byte drive values.

## Release

The repo went public on August 30. On October 16 I tagged rmvci-core 0.1.0, with CI on Linux, macOS, Windows and Android. It isn't on crates.io yet. It's GPL-3.0 because it uses libMVCI's captured test vectors and I haven't asked the author about relicensing.

Only the core is released. The J2534 shared library and Android layer are in the repo as source. `rmvci-diag` (KWP2000 and UDS clients) and `rmvci-net` (the TCP bridge) are held back: diag doesn't yet implement the DTC service I learned about on the car, and the bridge listens on all interfaces with no authentication.

## Still to do

1. Publish to crates.io: core, then diag, then net.
2. Finish `rmvci-diag`: DTC read (`13`) and clear (`14`), freeze frames, active tests, then customisation writes behind a confirmation step. Turn the car captures into mock tests.
3. Bind the bridge to localhost by default.
4. `tsdiag`, a CLI that scans the car and decodes everything using the extracted JSON, is written but hasn't been run against the car.
5. Find the unmet precondition in the headless harness so it emits value polls, then the P4/P5 data-list path.
6. Extraction gaps: CAN-side scale factors are overridden by two tables I haven't decoded, and DTC clear has never been exercised.
7. Things I can only learn from the car: actuator-drive frames, since I've never fired a test, and security access (`27`), which I've never seen, so the immobiliser, theft and meter ECUs return nothing.
8. Upstream the libMVCI CAN branch, and relicense to MIT if its author agrees.

Everything here was done on my own car with my own cable, and the only writes to the vehicle were ones Techstream itself made.
