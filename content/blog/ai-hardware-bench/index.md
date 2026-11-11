+++
title = "Bench instruments as Claude Code skills"
description = "The oscilloscope, multimeter, USB power meter and power switches I've wrapped as Claude Code skills, and the problems I ran into with each."
date = 2026-10-17

[taxonomies]
tags = ["AI", "Claude Code", "hardware", "test equipment", "automation"]

[extra]
toc = true
+++

I've been wrapping my bench equipment as [Claude Code skills](https://code.claude.com/docs/en/skills) so an agent can flash a board, power-cycle it and take measurements without me reading a meter for it. A skill is a folder with a `SKILL.md` (when to use it, usage notes, warnings) and usually a script.

One example of what this allows: on a battery-powered recorder project I suspected current draw had crept up. An agent wrote a harness that went through the commit history and, for each commit, built it, flashed it, booted it, and logged 60 seconds of current on the USB meter. It needed some help. Cold-booting via smart plug left the board with no USB data link, so I moved it to a hub, and the agent cherry-picked a later startup fix onto old commits so they survived a warm reflash. The result was about 28 mA at the first commit and about 90 mA at HEAD, mostly from two commits that changed how often the firmware flushed its writes. Restoring the flush was a one-line change that brought HEAD to 32 mA.

| Device | What the skill does |
|---|---|
| OWON VDS1022I USB oscilloscope | Hardware-triggered captures, measurements, FFT, gapless recording, protocol decode, plots |
| OWON XDM1041 bench multimeter | Volts, amps, ohms, capacitance, frequency, temperature; set function and range; log to CSV/JSON |
| RuiDeng TC66C USB-C meter | Log voltage, current, power and energy for anything USB-powered |
| VIA VL817 7-port USB hub | Cut and restore VBUS on individual ports |
| Fugetek Matter USB switch | Cut USB power to one device over Wi-Fi |
| TP-Link Kasa smart plugs | Switch mains power; read wall power for large loads |

## OWON VDS1022I oscilloscope

The VDS1022I is a two-channel isolated USB scope: 25 MHz, 100 MS/s, 8-bit, no screen. The official software is a Windows app. florentbr's [OWON-VDS1022](https://github.com/florentbr/OWON-VDS1022) reimplements the protocol with a Python API. My skill vendors that driver and adds a CLI: `measure`, `capture`, `plot`, `spectrum`, `record` and `decode`, sharing one set of acquisition and trigger options.

```bash
# single-shot: 10 MS/s, catch one rising edge through 1.5 V, 20% pre-trigger
vds1022_cli.py plot --rate 10M --range1 5 --trigger 1 --level 1.5 \
    --sweep once --position 0.2 --timeout 30 --out shot.png

# glitch hunt: a pulse high for less than 2 us
vds1022_cli.py capture --rate 100M --range1 5 --trigger 1 --mode pulse \
    --condition 'rise<' --width 2u --level 1.5 --out glitch.csv
```

Triggering stays in hardware. `--sweep once` arms the FPGA and waits, and if nothing fires before `--timeout` the CLI exits with code 2. `plot` writes a PNG the agent can look at, which is how it usually notices it's on the wrong range.

I had an agent decompile OWON's software and check every register encoding against the open-source driver. It found three bugs, which I've submitted upstream:

- [#121](https://github.com/florentbr/OWON-VDS1022/pull/121): the slope trigger never fired. The hardware orders trigger types edge/slope/video/pulse, and the driver's constants were in a different order, so asking for slope programmed a video trigger.
- [#122](https://github.com/florentbr/OWON-VDS1022/pull/122): the fall conditions were off by one. The hardware skips value 3 in the condition field, so each "fall" condition landed one slot too low.
- [#123](https://github.com/florentbr/OWON-VDS1022/pull/123): a crash on numpy 2. Adding 128 to an `int8` array used to widen the type, and numpy 2 raises `OverflowError` instead, which broke `levels()`, `amp()` and `to_ttl()`.

Each fix was tested against the scope's 1 kHz probe-compensation output.

Two other things went into the skill notes. GND coupling isn't real: the decompiled app overwrites the samples with a flat line at the zero position, and on the hardware only the AC/DC bit does anything. For a zero reference, short the probe to its ground clip. And the external trigger input is unreliable. Fed clean edges from an RP2040, it caught about 80% of them (CH1 caught 100% in the same test), and it ignores falling edges. No setting I tried changed that. The misses are independent per edge, so a burst of three edges gets to about 99.7% if you control the source. Otherwise the skill tells the agent to use a channel trigger.

## OWON XDM1041 multimeter

The XDM1041 is a 5½-digit bench meter that speaks SCPI over USB serial. The script works as a CLI and a library: `-f vdc`, `-f res` or `-f cont` picks the function, `-r` sets the range, `--rel` zeroes lead resistance, and `-n`/`-d` log a number of samples or a duration as CSV or JSONL. Each reading comes back as a float in base SI units plus the meter's display string.

The first version got about one reading per second because of `serial.read(64)`: SCPI replies are shorter than 64 bytes, so every query waited out the 1-second timeout. With `read_until(b'\r\n')` a query takes about 4.5 ms.

Some quirks: the meter sends Ω as UTF-8 from one query and as an old two-byte encoding from another, so the driver normalizes both. In thermocouple mode it stalls for about 3 seconds every 10 minutes. And with the function set to mA and the lead in the 10 A jack, it reads zero while current flows. The agent can't see the probes, so the skill tells it to ask me what's wired up.

## TC66C USB-C power meter

The TC66C is a small inline USB-C meter, and it gets used the most because most of what I build is USB-powered. The skill logs voltage, current and power, then prints mean, min, max and integrated energy.

On the stock firmware, polling faster gives worse data. Each reading is an AES-encrypted packet, and the encryption takes time away from measuring on the single-core MCU. At 100 ms you get the same stale value five times in a row. The skill enforces a 0.2 s minimum interval and has a `--dedupe` flag.

I also wrote a replacement firmware for this meter, which is [a separate post](@/blog/tc66-custom-firmware/index.md).

## Per-port USB power with uhubctl

[uhubctl](https://github.com/mvp/uhubctl) switches power on individual ports of hubs that support it.

My first hub was a Rosonway 7-port listed in uhubctl's supported-hubs table. It had two Realtek RTS5411 chips, both advertising per-port power switching, and it appeared to work: devices disappeared from `lsusb`, and an SD card reader dropped from 22 mA to under 1 mA. But the TC66 still showed 5 V on the "off" port. I plugged in a USB battery pack as a load that doesn't care about data, and it drew about 200 mA through every kind of "off", including a raw `ClearPortFeature(PORT_POWER)` sent to the hub chip. The supported entry turned out to be a VIA-based version of the same model.

The replacement is a 7-port hub built from two daisy-chained VIA VL817 chips, and switching a port off takes the inline TC66 to 0.000 V and 0.0 mA. The first agent to test it nearly concluded it was fake too, because it had switched a port on a different VIA chip with the same USB IDs, inside an Anker hub downstream of the meter.

So the skill works like this:

- It finds the bench hub by topology, not ID: a daisy-chained pair of 4-port VL817s, with ports named `a1..a4` and `b1..b4`. It refuses to switch `a3`, the internal link to chip B.
- It switches both halves of each port. Each physical port belongs to a USB2 hub and a USB3 hub, and VBUS stays up while either is powered. It checks the whole tree on every command, and if a sub-hub failed to enumerate it prints `HUB IN A BAD STATE` instead of silently doing nothing.
- A power cut is only confirmed by 0 V on a meter, not by a device leaving `lsusb`. The TC66's data cable goes on a different port so it keeps reporting.

```
$ hub_power.py status
chip a
  a3  on   [internal link to chip b — off powers down all b* ports]
  a4  on   0403:6001  FTDI FT232R USB UART
  ...
$ hub_power.py cycle a4 -t 5
```

## Matter USB switch

For devices away from the hub there's a Fugetek Matter USB switch, a USB-A passthrough that cuts power to whatever is plugged into it. Matter commands need an encrypted session from a controller on the device's fabric, and mine is the Matter server on my Home Assistant box. The skill calls that server's WebSocket API directly, sends an OnOff command, and prints the state the device reports.

The switch never appears in `lsusb` and has no metering, so the agent can't tell what's plugged into it. The skill only allows using it after I've said what's downstream, and forbids toggling it to find out. The current downstream device is one line in `SKILL.md`.

## Kasa smart plugs

TP-Link Kasa EP25 plugs switch mains power, for a Pi's supply or a dev board that needs a true cold boot. The skill takes a plug number matching the label on the physical plug, with no default. Commands are `status`, `on`, `off`, `reset` (with an adjustable off time so supply capacitors drain) and `energy`.

The plugs log in with my TP-Link account, and the original script had the password hard-coded. Since agents read skill folders into context, the credentials now live in a mode-0600 file outside the skill.

The EP25's energy reading can't see small loads. With the TC66 showing about 130 mW on a board, the plug read `0.0 W` across eight samples. The skill limits the plug's energy reading to appliance-sized loads and points anything USB-powered at the TC66.

## Sharing the bench between agents

With several agents using the same hardware, one will want to power-cycle a board while another is mid-capture. `agent-coord` is a small broker on a local Unix socket. It has leases (exclusive holds with a time limit, so a crashed agent can't hold a device forever), latched signals (which still work if the other agent finished first), and small message channels.

```bash
coord.py acquire scope --owner ripple-test --ttl 600
# ... capture ...
coord.py release scope --owner ripple-test
```

It's a convention and not an interlock, so it only works for agents that ask first.

## Notes

The scripts were the smaller part of the work. Most of it was getting each `SKILL.md` to describe what the tool can't do: the trigger that catches 80% of edges, the coupling mode that's drawn in software, the plug that reports zero watts, the hub that reports a port as off with the power still on. An agent trusts its tool output, so those limits need to be written down where it will read them.
