+++
title = "Custom firmware for the TC66C USB power meter"
description = "Reverse-engineering the RuiDeng TC66C's protocol, firmware encryption and PCB, and writing a replacement firmware in Rust that can trigger USB-PD, QuickCharge and Samsung AFC over USB."
date = 2026-09-21
updated = 2026-10-16

[taxonomies]
tags = ["reverse engineering", "Rust", "firmware", "USB-PD", "hardware"]

[extra]
toc = true
+++

The RuiDeng TC66C is a USB-C inline power meter with a small colour screen. It reports voltage, current, power and cumulative energy for whatever is plugged through it, and costs about $30. I bought one to measure the power draw of a recorder board.

I wanted to script it from my PC: poll it quickly, log the readings, and have it trigger a charger to 9 V or 12 V, which it can do from its buttons. The trigger turned out not to be available over USB at all, and getting it there meant replacing the firmware. Most of the work was August 15–26, 2026, with more added through mid-October.

The firmware is open source (MIT): **[github.com/FletcherD/tc66-custom-fw](https://github.com/FletcherD/tc66-custom-fw)**. There are prebuilt releases, a flashing tool, and a way back to stock.

## The stock protocol

The TC66C appears on Linux as a USB CDC serial port. To find out what it accepts, I looked at the vendor's Windows and Android apps.

The Windows app is .NET, with its dependencies bundled by Costura and its strings encrypted by ConfuserEx, so `strings` finds no commands. ILSpy resolves the string encryption automatically. All serial I/O is in one class, and it either sends one of a few fixed 5-character strings or streams a firmware image in 64-byte chunks.

The Android app is packed with 360 Jiagu. The `classes.dex` in the APK is a stub, the real code is decrypted into memory at runtime, and the packer wipes the DEX magic in memory to make dumping harder. On a rooted phone I found the real DEX in `/proc/<pid>/maps` by its size instead of its magic, dumped it from `/proc/<pid>/mem`, and wrote the magic back. The app is BLE-only and its commands mirror the USB ones.

The full command set is seven unterminated 5-byte ASCII commands:

| Command | Reply | Purpose |
|---|---|---|
| `getva` | 192 bytes, encrypted | poll all measurements |
| `rotat` | none | rotate the screen 180° |
| `nextp` / `lastp` | none | next / previous screen |
| `gtrec` | 8-byte records + `reck` | dump the internal ~1 Hz datalogger |
| `query` | `firm…` or `boot` | is the app or the bootloader running |
| `update` | `uprdy`, then `OK` per chunk | (bootloader) start flashing |

Over BLE they're the same with a `b` prefix and CRLF. There's no command for PD, QuickCharge, AFC or FCP/SCP. The device has the fast-charge triggers, but they're only reachable from the buttons.

Other things I found while testing the protocol:

- The `getva` reply is three 64-byte blocks, AES-256-ECB encrypted with a static key, with measurements at fixed little-endian offsets. Two keys circulate online and only one (the one from rdserialtool) works on my unit.
- The firmware reads one USB packet at a time and compares the first 5 bytes to its command table. There's no ACK or NAK, and a command split across packets is ignored.
- Polling faster returns fewer distinct values. The measurement loop refreshes at about 2–3 Hz, and each `getva` reply costs about 60 ms of AES and transmission on the same core. The best poll interval is about 0.22 s.
- `gtrec` records aren't encrypted. Logging runs whenever there's voltage on the port, and there's no command to start, stop or clear it.
- The meter advertises over BLE under its module's name, `BT24-M`, and the signal is weak: about −76 dBm from a few inches away.

## Getting the firmware key

Firmware updates are AES-256-ECB encrypted and decrypted by the bootloader as it flashes. The key isn't in the host software, which sends the `.bin` unmodified. The MCU is a GigaDevice GD32F150G8: Cortex-M3, 64 KB flash, 8 KB SRAM. I tried two ways of getting the key at the same time.

### Power analysis (didn't work)

I spent about five days on correlation power analysis. I control the ciphertext, and the decrypt happens about 9.1 ms after each chunk, so it looked like a good target. I used a ChipWhisperer-Nano on the supply shunt, a hand-wound EM probe through a low-noise amplifier, a Hantek 6022BE, and finally an OWON VDS1022I at 100 MS/s on both the shunt and the EM probe.

It never converged. At one point a machine-learning attack named key byte 0 as `0x72` with a plausible p-value, but a permutation test (shuffling the labels) produced the same strength of result, and a fresh capture named a different value.

After I had the key by the other route, I found out the attack couldn't have worked as built. I had assumed AES-128 when it's AES-256, and my check assumed the fill block decrypts to `0xff`×16 when it's actually `0x00`×16.

### Reading it from SRAM

For SWD access I soldered wires to the board's 5-pad header and NRST and connected them to a Raspberry Pi Pico running a bit-banged SWD engine.

The chip is at RDP level 2. Direct flash reads fault. The published trick of using DMA as a bus master to copy flash, which worked on a related chip, returned zeros here. A stub loaded into SRAM faults on its first flash read. Attaching SWD also halts the core, which I mistook for firmware crashes several times.

RDP2 doesn't protect SRAM, though, and SWD can read SRAM while the firmware runs. The bootloader has to hold the key and the decrypted data in SRAM while it decrypts. So I booted into the bootloader (hold K1 while plugging in), sent `update`, streamed one encrypted 64-byte block from a real firmware image, and dumped the 8 KB of SRAM. Diffing against a dump of the idle bootloader left about 294 changed words, including a 32-byte value at offset 0x2c8:

```
381255645a6b7aaabc6d5877f5c1fbff9805d1ac3c8335661a610c7ff73a638c
```

It decrypts every firmware version I have to a valid Cortex-M3 image.

## The stock firmware in Ghidra

The app is linked at `0x08002c00`, directly after the 11 KB bootloader, not at `0x08000000`. Loaded at the wrong base, the Thumb code still disassembles because it's mostly PC-relative, but the absolute pointers (vector table, string tables, jump tables) are all off by 0x2c00 and almost no cross-references form.

The command dispatcher is a chain of first-byte compares, which is why searching for command strings finds nothing. It confirmed the seven commands, and that nothing in the USB or BLE command path reaches the fast-charge code.

Three AES-256 keys sit together at `0x080098c3`: two UID-signature keys and the `getva` key. Before its main loop, the app encrypts the chip's factory UID and compares the result with a stored signature, and loops forever on a mismatch. A modified app on the same chip still passes, because the signature is in a config page that flashing the app doesn't touch.

The bootloader (v1.06) reads K1 on PF0 at power-on. Released, it validates the app and jumps to it. Held, it enters serial update mode. It has six commands, including `uplogo`, `recali`, `writsn`, and a `gtsub` that returns the chip UID. Two things about it matter for custom firmware:

- It doesn't check the image at all: no CRC, magic number, version or signature.
- The write offset only increases from the app base, so an update can't overwrite the bootloader or the UID signature. A bad flash is always recoverable by holding K1 and flashing stock.

So anyone with the image key can flash any Cortex-M image of 52 KB or less.

## The board

I needed the debug pads and the full pin map, which meant identifying the parts and tracing a dense two-sided board of about 22.6 × 40.2 mm.

The parts: the GD32F150G8 (QFN-28), a DX-BT24-M BLE module, a BL24C256 I²C EEPROM, a TI INA226 current/voltage monitor, a 15 mΩ sense shunt, an M5633B 3.3 V LDO, a FUSB302 USB-PD PHY (which I only found when an I²C scan got an ACK at 0x22), and a 0.96" ST7735S TFT. There's no load switch, so firmware can't turn VBUS off.

My first imagery was about 34 frames from a freehand microscope video. OpenCV and Hugin stitched them wrongly, because the repeated silkscreen produces confident false matches. One frame matched with 439 inliers and was a centimetre off. I wrote a small browser tool where I place each frame roughly by hand and a SIFT step refines it, rejecting any match that would move a frame more than half its width.

Later I removed all the parts with hot air and scanned the bare PCB on a flatbed at 2400 dpi. I registered the two sides to about 2.4 µm RMS using the mounting holes, traced the board into KiCad over the scans, and checked ambiguous nets with a multimeter.

| Function | Pins |
|---|---|
| USB D+ soft-connect | PA9 (also the backlight enable) |
| I²C0 (INA226 0x4A, EEPROM 0x50, FUSB302 0x22) | SCL=PB6, SDA=PB7 |
| On-chip ADC | ch0=PA0 NTC, ch1=PA1 D+, ch9=PB1 D− |
| TFT (ST7735S over SPI0) | SCK=PA5, MOSI=PA7, DC=PA4, RST=PA3; backlight PWM on PB3 |
| Buttons | K1=PF0 (bootloader), K2=PF1 (factory reset) |
| Charge trigger (D+/D− drive) | D+ via PA13/PA14, D− via PB4/PB5 |
| BLE module | USART1: TX=PA2, RX=PB0 |

From the stock firmware I also got the behaviour behind the pins. The INA226 is used as a raw ADC (config `0x41A7`) with averaging and calibration done in software from per-unit constants in the EEPROM. The NTC is 10 kΩ, β=3450, on the bottom of an 11.5 kΩ divider. The EEPROM holds an energy ring at 0x0000, calibration at 0x2000/0x2200, and the datalogger ring at 0x2400. The display init sequence and gamma tables came from there too.

## Patching the stock firmware

Before writing new firmware I patched the vendor's, to check that I understood it.

`fwtool.py` decrypts, patches, re-encrypts and verifies. It checks that `encrypt(decrypt(vendor.bin)) == vendor.bin` for every version, and patches are `(offset, expected_old_bytes, new_bytes)` tuples that won't apply if the old bytes don't match. The first patch changed the on-screen version string from `V1.18` to `V1.19`.

The first real patch fixed the polling problem. `getva` sends its three 64-byte blocks with a `delay_ms(10)` busy-wait between them, which takes loop iterations away from the sampler. The patch replaces the delay with a bounded spin on the USB endpoint's transmit-status flag. On hardware it gave 40 of 40 clean decodes and about 99% complete replies under back-to-back polling. I considered moving the sampler into a timer interrupt and decided not to. The only spare timer runs the 1 Hz energy integration, the sampler functions aren't re-entrant, and the ADC only refreshes at 2–3 Hz anyway.

The second patch added an `updat` command that lets the running app reflash itself over USB without the bootloader button. The app runs from the region being erased, so the flasher runs from SRAM, polls the USB peripheral registers directly, and calls the bootloader's flash routines by absolute address. Chunks are sent as plaintext. It updates in about a second, roughly 9× faster than the bootloader path.

## The new firmware

The custom firmware is `no_std` Rust. It goes in the 52 KB app slot and leaves the stock bootloader in place, so recovery still works. It enumerates as the same USB device and answers the stock commands.

### Booting and self-flashing

The first milestone was booting my own code, enumerating as the stock CDC device, answering `query`, and reflashing itself with `updat`, so development never needs the button. What the bootloader requires of the app:

1. The initial stack pointer has to be `0x20000E40`. Top-of-RAM faults during the bootloader's hand-off, and I never found out why.
2. The app has to set the vector-table offset register to `0x08002c00` itself.
3. PA9 is the USB D+ soft-connect. The bootloader hands off with the bus still connected, so the app has to drive PA9 low, bring USB up, then drive PA9 high, or the host never re-enumerates.
4. It needs a complete vector table and a cleared NVIC.

USB uses `usb-device`, `usbd-serial` and `stm32-usbd`, since the GD32's USB peripheral is the STM32F0/F1-style one. The SRAM flasher erases the flash it was loaded from, so it uses raw pointers only, avoids anything that could emit a `memclr` call into flash, and calls the bootloader's flash functions through absolute-address function pointers. A build step disassembles the flasher and fails the build if any instruction references app flash.

One bug here: I had placed the flasher's linker section with `INSERT AFTER .data`. The Rust runtime's startup copy extends over sections after `.data` on the assumption that they're contiguous, and the flasher's run address is somewhere else, so every boot copied about 6 KB of flash across low RAM. That was harmless until I added the peripheral-access crate, whose one static got overwritten. The section now goes after `.uninit`, and another build check asserts the memory sections land in real RAM.

### Measurement

I brought up each peripheral behind a host-triggered command first, with bounded waits and staged error codes, and only moved it to the always-on path once it worked. That way a mistake can't break booting.

The I²C scan found the INA226 and the EEPROM. I dumped the whole EEPROM twice as a calibration backup before writing anything to it. The calibration model from the stock firmware (integer math with per-unit gain, offset and zero-reference constants) reproduced a known 5.2084 V exactly from the raw registers. `getva` replies are built from a template with the live fields patched in, the three per-block CRC16s recomputed, and the result AES-encrypted. Existing host tools work with it unchanged.

The on-chip ADC's clock comes from a dedicated 14 MHz oscillator that's off at reset, and the ADC's self-calibration hangs without it. My first channel map was also partly wrong, because I took the stock code's inserted-group rank numbers for channel numbers. The pin I'd been reading as D+ was the Bluetooth module's UART output. The temperature calculation uses the stock β model in integer fixed point, and a host-side test checks it against the stock floating-point formula for every possible input.

### Sampling, display, storage

- A timer interrupt reads the INA226 every 2.24 ms, getting both shunt and bus voltage each tick. That catches about 98% of the conversions the INA226 makes and doesn't depend on host polling. The averaging window and screen refresh rate are settings.
- A plaintext `rawva` command with a sequence number, because an encrypted `getva` was costing about 20 ms of software AES. It streams at 700 polls per second without missing sampler windows.
- The device is a composite USB device that also emulates a ChargerLAB POWER-Z KM003C on a vendor interface, so the mainline Linux `powerz` hwmon driver binds and `sensors` works. The KM003C's graph mode streams at 2, 10, 50 or 1000 samples per second, and the [km003c-rs](https://github.com/okhsunrog/km003c-rs) CLI and GUI work with it unmodified.
- The ST7735S display is driven over SPI with DMA, pumped from the main loop so it doesn't block USB or the sampler. A static screen left on overnight caused image persistence on the LCD, so the screen turns off after 15 seconds without a change in input.
- Six screens, stepped with K1: V/A/W, energy and run time, temperature and D±, a scrolling V/I plot (using the ST7735S's hardware vertical scroll), USB-PD info, and charger detection. Holding K2 resets the energy counters.
- Settings, the energy ring and the datalogger are stored in the stock firmware's EEPROM layout, so they carry over when switching between stock and custom. The EEPROM driver verifies every page and refuses to write the factory calibration words.
- A hardware watchdog of about 13 s.

### GD32F150 quirks

I keep a `QUIRKS.md` of reproducible GD32F150 misbehaviour, using the stock firmware as a reference for whether a problem is the chip or my code.

- While the USB host is NAK-polling an endpoint, the CPU's reads of unrelated peripheral registers stall for 1–5 ms each. Under a NAK storm the polled main loop dropped to about 70 iterations per second. I fixed it with pre-armed replies, a core-internal timebase, and USB handled in an interrupt.
- USB completion events get lost, and reading the status register at ISR entry consumes events. With that read in place, 0 of 401 replies produced a completion callback. Without it, half did. I ended up copying the stock firmware's approach, a drain loop that re-checks per-endpoint flags.
- Only the first 32 KB of flash is zero-wait. Above that, instruction fetches are about 10× slower, and my linker had put the AES code up there. A linker script that pins interrupts, USB, AES and the main loop below 32 KB took `getva` from 48.6 ms to 2.3 ms. To find what to pin, I sampled the interrupted PC from the exception stack frame on every sampler tick into a ring in SRAM.

Because of RDP2, SWD halt, reset and step lock the core, so `probe-rs`, RTT and `defmt` don't work. Debugging is done with marker values in SRAM and the dev commands over serial.

### Stack and flash budget

A stack overflow on the boot path hardfaults before the D+ pull-up, and the board looks dead until it's recovered with K1. That happened once. Now every build runs a script that computes a worst-case stack depth from the disassembled call graph, including the deepest interrupt chain, and checks it against the RAM budget. Large statics such as the AES key schedule and a PD capture ring live in a section in the 1 KB of RAM above the stack.

Flash is full, with about 150 bytes spare in the release build. Size-optimised builds with fat LTO took it from about 49.5 KB to 35.6 KB, and later features used that up. The last large saving was replacing the AES crate with a hand-written AES-256, which saved 4.3 KB and made `getva` three times faster.

## Fast-charge triggers

This was the original goal. The triggers went in over one day, and the PD sniffer and charger detection followed in October. Everything except FCP/SCP was tested against real chargers.

### USB-PD and PPS

The FUSB302 handles the Type-C physical layer, including CRC and the automatic GoodCRC reply. My `pd.rs` implements a minimal PD 2.0/3.0 sink: detect attach and orientation, collect the source's capabilities, and negotiate a contract. Four host commands (`pdctl`, `pdsta`, `pdcap`, `pdreq`) arm it, report state, list the source's profiles and request one.

It works against an Apple 96 W charger at 5, 9, 15 and 20 V. There were two bugs along the way. A source that was plugged in before the sink was armed has already given up sending capabilities, so arming now drops the pulldowns for about 600 ms to look like a fresh attach. And a timeout compared against a timestamp captured before several milliseconds of I²C, so the subtraction wrapped and the `Accept` timeout fired immediately.

For PD 3.0 and PPS, the sink starts at revision 2.0 and only moves to 3.0 after the source declares it, because a 3.0 reply to a 2.0 message made the Apple charger reset the link. PPS contracts expire unless re-requested within about 10 s, so the firmware re-sends the request every 8 s. PPS stepping across 3.3, 5, 9 and 11 V tracked within about 50 mV.

The firmware also exports every PD message it sends or receives in the KM003C wire format over the POWER-Z endpoint, so KM003C tools can show the PD traffic.

### PD sniffer

In sniffer mode the meter doesn't present as a sink. It records what a charger and device say to each other through it: source capabilities, requests, vendor messages, the cable's e-marker on SOP′/SOP″, attaches and detaches, hard resets, and packets with bad CRCs. A host tool, `pdcap.py`, decodes the messages using layouts from the USB PD R3.2 spec.

At the FUSB302's default BMC slicer threshold, one VDM went missing from every capture, and adjusting the threshold fixed that. My unit also has a hardware fault: the OUT port's CC2 contact is nearly open, so sink attach and hard reset only work in one orientation.

### QuickCharge 2.0 and 3.0

QuickCharge signals with DC voltages on D+/D−, so it's a separate GPIO-only engine driving the resistor network on those four pins. QC2.0 measured 5.11, 9.13 and 12.14 V. QC3.0's continuous mode steps ±200 mV per pulse: from a base of 5.31 V, ten steps up gave 7.30 V, and ten down returned to base. Pulses need about 20 ms between them or they merge. The D+ drive pins are also SWDIO/SWCLK, so SWD doesn't work while a QC trigger is active.

### Samsung AFC

AFC is a single-wire serial protocol on D−. I implemented it from my reverse-engineering notes as 152 µs NRZ bits, MSB first, odd parity, with a start pattern, and the charger didn't respond. This one took a while to debug:

1. My scope showed a flat line during transmission because the probe wasn't referenced to the driven node. The firmware's own ADC showed the pin was being driven.
2. With the probe fixed, the timing was wrong. The bit delays were a busy loop in slow flash, above the 32 KB boundary, so each 152 µs bit took about 1090 µs. Using the SysTick counter fixed that, and the scope decoded my output correctly. The charger still didn't switch.
3. I added an open-drain acknowledgement bracket around each byte, with no change.
4. A capture of the stock firmware's output made me think the encoding was pulse-width and not NRZ, and I started rewriting the engine. A higher-sample-rate capture showed it was NRZ after all. The earlier reading was an undersampling artifact.

The encoding had been right since step 2. Three other things were missing. The protocol is closed-loop: the master sends a ping, then reads the pin to follow the charger's response ping before each byte. The entry sequence has to release the lines to high impedance and let the charger source the D− level. And the bit timing has to run with interrupts masked so the sampler interrupt can't stretch a bit. With those, `afctr` takes a Samsung charger from 5 V to a measured 8.92 V.

### Huawei FCP/SCP

FCP and SCP use the same physical layer as AFC, with multi-byte register packets and a Huawei CRC-8. I don't have a Huawei charger, so this code is untested and marked as such, with the assumptions I couldn't check noted in the code. The main one is whether parity is per byte or across the whole packet.

### Charger detection

The detection screen shows what a charger supports without going above 5 V. One K2 press reads the Type-C current advertisement, checks for Apple and Samsung D± dividers, probes QuickCharge and AFC, and lists every PD offer. It won't start if something is plugged into the other port.

### BLE

I reimplemented the stock BLE transport for the BT24 module over its UART, with DMA for receive and transmit so it uses no interrupts. The module is on USART1 (TX=PA2, RX=PB0). The stock firmware never configures the module, so it runs at its default 9600 baud. It answers AT queries but rejects persistent writes with `ERROR=104`. The stock `b`-prefixed commands work as before.

## Current state

The firmware does what I wanted: USB-PD, PPS, QuickCharge 2.0/3.0 and Samsung AFC triggers over USB, plus sampling that doesn't depend on polling, POWER-Z emulation, the PD sniffer, charger detection and BLE. The repo has CI that builds and runs the build checks on each push. Each release is one encrypted image in the bootloader's format plus a flashing script, `tc66flash.py`.

Going back to stock always works, because nothing here touches the bootloader. Hold K1 and run `tc66flash.py stock`, which downloads RuiDeng's V1.18 from their server.

Everything here was done on hardware I own. The vendor's firmware images and apps aren't redistributed.
