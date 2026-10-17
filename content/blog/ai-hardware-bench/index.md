+++
title = "An AI-drivable hardware bench: the instruments and power switches my agents can use"
description = "A tour of the oscilloscope, multimeter, USB power meter, and power switches I've wrapped as Claude Code skills, so an agent can measure and power-cycle real hardware on its own."
date = 2026-10-17

[taxonomies]
tags = ["AI", "Claude Code", "hardware", "test equipment", "automation"]

[extra]
toc = true
+++

*A tour of the instruments and power switches on my bench that an AI agent can drive by itself: what each device is, what its skill exposes, and what went wrong along the way.*

---

In embedded work the slow part is rarely the code. It's the loop around it: flash, power-cycle, wait for boot, read a meter, write the number down, repeat. An AI coding agent will happily write the firmware, but without the hardware it has to stop and ask me to read the meter.

So I've been wrapping my bench equipment as [Claude Code skills](https://code.claude.com/docs/en/skills). A skill is a folder with a `SKILL.md` (a description of when to use it, plus usage notes and warnings) and usually a script. The agent reads the notes, runs the script, and gets numbers back.

The example that sold me: on a battery-powered recorder project I suspected current draw had crept up at some point. An agent wrote a harness that went back through the commit history and, for each commit, built it, flashed it, booted it, and logged 60 seconds of current on the USB meter. It wasn't completely hands-off. Cold-booting via smart plug left the board with no USB data link, so I moved it to a hub, and the agent cherry-picked a later startup fix onto old commits so they survived a warm reflash. But it came back with a curve: about 28 mA at the first commit, about 90 mA at HEAD, mostly from two commits that changed how often the firmware flushed its writes. Putting the flush back was a one-line change that brought HEAD to 32 mA. Then I went to bed and left it running an overnight test matrix.

| Device | What the agent can do with it |
|---|---|
| OWON VDS1022I USB oscilloscope | Hardware-triggered captures, measurements, FFT, gapless recording, protocol decode, plots it can look at |
| OWON XDM1041 bench multimeter | Volts, amps, ohms, capacitance, frequency, temperature; set function and range; log to CSV/JSON |
| RuiDeng TC66C USB-C meter | Log voltage, current, power and energy for anything USB-powered |
| VIA VL817 7-port USB hub | Cut and restore VBUS on individual ports |
| Fugetek Matter USB switch | Cut USB power to one device over Wi-Fi |
| TP-Link Kasa smart plugs | Switch mains power; read wall power for big loads |

## The OWON VDS1022I oscilloscope

The VDS1022I is a cheap two-channel isolated USB scope: 25 MHz, 100 MS/s, 8-bit, no screen. The host loads an FPGA image at power-up, and the official software is a Windows app. florentbr's [OWON-VDS1022](https://github.com/florentbr/OWON-VDS1022) reimplements the protocol with a Python API. My skill vendors that driver and puts a CLI on top: `measure`, `capture`, `plot`, `spectrum`, `record` and `decode`, all sharing one set of acquisition and trigger options.

```bash
# single-shot: 10 MS/s, catch one rising edge through 1.5 V, 20% pre-trigger
vds1022_cli.py plot --rate 10M --range1 5 --trigger 1 --level 1.5 \
    --sweep once --position 0.2 --timeout 30 --out shot.png

# glitch hunt: a pulse high for less than 2 us
vds1022_cli.py capture --rate 100M --range1 5 --trigger 1 --mode pulse \
    --condition 'rise<' --width 2u --level 1.5 --out glitch.csv
```

The key choice is that triggering stays in hardware. `--sweep once` arms the FPGA and waits, so nothing slips past between polls. If nothing fires before `--timeout`, the CLI exits with code 2, so a bad trigger setup is a clear failure rather than a hang. `plot` writes a PNG the agent can look at, which is the fastest way for it to notice it's on the wrong range.

I had an agent decompile OWON's official software and check every register encoding against the open-source driver. It found three bugs, which I've submitted upstream:

- **[#121](https://github.com/florentbr/OWON-VDS1022/pull/121): the slope trigger never fired.** The hardware orders trigger types edge/slope/video/pulse. The driver's constants used a different order and were written to the register as-is, so asking for a slope trigger programmed a video trigger.
- **[#122](https://github.com/florentbr/OWON-VDS1022/pull/122): the fall conditions were off by one.** The hardware skips value 3 in the condition field, so each "fall" condition landed one slot too low. "Fall longer than" never fired, and the other two got the wrong comparison.
- **[#123](https://github.com/florentbr/OWON-VDS1022/pull/123): crash on numpy 2.** Adding 128 to an `int8` array used to quietly widen the type. numpy 2 raises `OverflowError` instead, which broke `levels()`, `amp()` and `to_ttl()`.

Each fix was tested against the scope's 1 kHz probe-compensation output, with cases that should fire and cases that shouldn't.

Two more findings went into the skill notes. First, **GND coupling is fake.** The manual says Ground "disconnects the input signal", but the decompiled app just overwrites the samples with a flat line at the zero position. On the hardware, only the AC/DC bit of the coupling field does anything. For a real zero reference, short the probe to its ground clip. Second, **the external trigger input is unreliable.** Fed clean edges from an RP2040, it caught about 80% of them, against 100% for the same loop on CH1, and it ignores falling edges completely. No setting I swept made any difference. The misses are independent per edge, so if you control the source, a burst of three edges gets you to about 99.7%. Otherwise the skill steers the agent to a channel trigger.

## The OWON XDM1041 bench multimeter

The XDM1041 is a 5½-digit bench meter that speaks SCPI over USB serial. The skill's script works as a CLI and as a library: `-f vdc`, `-f res` or `-f cont` picks the function, `-r` sets the range, `--rel` zeroes out lead resistance, and `-n`/`-d` log a set number of samples or a set duration as CSV or JSONL. Each reading comes back as a float in base SI units plus the meter's own display string, and the agent is told to do its math on the float.

The first version only managed about one reading per second. The culprit was `serial.read(64)`: SCPI replies are shorter than 64 bytes, so every query waited out the full 1-second timeout. With `read_until(b'\r\n')`, a query takes about 4.5 ms.

The meter has quirks of its own. It sends Ω as UTF-8 from one query and as an old private byte pair from another, so the driver normalizes both. In thermocouple mode it stalls for about 3 seconds every 10 minutes. The biggest hazard is physical: with the function set to mA and the lead in the 10 A jack, the meter reads zero while current flows. The agent can't see the probes, so the skill tells it to ask me what's wired up before trusting a reading.

## The TC66C USB-C power meter

The TC66C is a small inline USB-C meter, and it's the instrument my agents use most, because most of what I build is USB-powered. The skill logs voltage, current and power, then prints mean, min and max along with the integrated energy. That's exactly what you need to answer "did this change make idle current better or worse?"

On the stock firmware, polling faster makes things *worse*. Each reading is an AES-encrypted packet, and the encryption eats the single-core MCU's time for actual measuring. At 100 ms you get the same stale value five times in a row. So the skill enforces a 0.2 s floor, adds a `--dedupe` flag, and tells the agent that anything faster than a few tenths of a second needs the scope.

I went much further with this meter and wrote a replacement firmware for it. That's [its own long post](@/blog/tc66-custom-firmware/index.md).

## Per-port USB power switching with uhubctl

[uhubctl](https://github.com/mvp/uhubctl) switches power on individual ports of hubs that support it, which makes a hub port the cheapest way to cold-boot a wedged board. It's also where I got fooled most convincingly.

My first hub was a Rosonway 7-port listed in uhubctl's supported-hubs table. Inside were two Realtek RTS5411 chips, both advertising per-port power switching, and it seemed to work. Devices vanished from `lsusb`, and an SD card reader dropped from 22 mA to under 1 mA. But the TC66 still showed 5 V on the "off" port, and the reader's D+ line sat at 3 V, which an unpowered device can't do. So I plugged in a USB battery pack, a pure charging load that doesn't care about data. It drew about 200 mA through every kind of "off": uhubctl, the USB2 and USB3 halves switched together, even a raw `ClearPortFeature(PORT_POWER)` sent straight to the hub chip. The hub reported the port as off and VBUS never dropped. The supported entry turned out to be a VIA-based version of the same model. Mine had Realtek silicon, and its switches did nothing.

The replacement is a 7-port hub built from two daisy-chained VIA VL817 chips, and switching a port off takes the inline TC66 to **0.000 V, 0.0 mA**. Even so, the first agent to test it nearly called it fake too. It had switched a port on a *different* VIA chip, the one inside an Anker hub downstream of the meter, which has the same USB IDs.

That shaped the skill:

- **It finds the bench hub by shape, not ID**: a daisy-chained pair of 4-port VL817s, with ports named `a1..a4` and `b1..b4`. It refuses to switch `a3`, the internal link that feeds chip B.
- **It switches both halves of each port.** Each physical port belongs to a USB2 hub and a USB3 hub, and VBUS stays up while either half is powered, so the script switches the USB3 twin along with the USB2 port. It also checks the whole tree on every command. If a sub-hub failed to enumerate, there's no twin to switch and `off` would silently do nothing, so it prints `HUB IN A BAD STATE` instead.
- **A power cut means 0 V on a meter.** A device leaving `lsusb` doesn't count. The TC66's data cable goes on a different port, so the meter keeps reporting while the rail it's measuring dies.

```
$ hub_power.py status
chip a
  a3  on   [internal link to chip b — off powers down all b* ports]
  a4  on   0403:6001  FTDI FT232R USB UART
  ...
$ hub_power.py cycle a4 -t 5
```

## The Matter USB switch

For devices away from the hub there's a Fugetek Matter USB switch, a USB-A passthrough that cuts power to whatever is plugged into it. You can't control Matter devices directly over the LAN. Every command needs an encrypted session from a controller on the device's fabric, and mine is the Matter server on my Home Assistant box. So the skill bypasses Home Assistant and calls that server's WebSocket API, sends an OnOff command, and prints the state the device reports back.

The interesting part of this skill is a rule. The switch is invisible to the agent: it never appears in `lsusb`, and it has no metering, so nothing reveals what's plugged into it. The skill only lets the agent use the switch after I've said what's downstream, and it forbids toggling the switch to find out. The current downstream device is one line in `SKILL.md`, and agents update that line when I tell them it changed.

## The Kasa smart plugs

TP-Link Kasa EP25 plugs switch mains power, for example to a Pi's supply or to a dev board when I want a true cold boot. (As the bisect showed, a board on a wall supply loses its serial console. When I need both, I use the hub.) The skill takes a plug number that matches the label on the physical plug, with no default, so an unnumbered "the plug" is an error rather than a guess. Commands are `status`, `on`, `off`, `reset` with an adjustable off time so supply capacitors drain, and `energy`.

Two lessons went in. The plugs log in with my TP-Link account, and the original script had the password hard-coded. An agent reads skill folders into its context, so the credentials now live in a mode-0600 file outside the skill, and the skill tells agents never to print them. The other is about `energy`: the EP25 can't see small loads. With the TC66 showing about 130 mW on a board, the plug read a flat `0.0 W` across eight samples while mains voltage read normally. A confident-looking zero is the worst kind of answer to hand an agent, so the skill limits the plug to appliance-sized loads and sends anything USB-powered to the TC66.

## Sharing the bench between agents

With several agents driving the same hardware, they start to collide: one wants to power-cycle a board while another is halfway through a capture. `agent-coord` is a small broker on a local Unix socket that offers **leases**, which are exclusive holds with a time limit so a crashed agent can't lock a device forever. It also offers **latched signals**, which still work if the other agent finished first, and small message channels.

```bash
coord.py acquire scope --owner ripple-test --ttl 600
# ... capture ...
coord.py release scope --owner ripple-test
```

It's a convention, not an interlock, and it only stops agents that ask first. When the scope was busy once, I told the agent to stick to reading code and claim the scope through `agent-coord` before testing anything. It did.

## What I've learned

The scripts were the easy part. The real work was making each `SKILL.md` honest about what its tool can and can't do: the trigger that catches 80% of edges, the coupling mode drawn in software, the plug that reports zero watts, the hub whose "off" left the power on. An agent believes whatever its tools tell it, so most of the effort went into making sure the tools never mislead it, and that every power cut is checked on a meter rather than assumed.
