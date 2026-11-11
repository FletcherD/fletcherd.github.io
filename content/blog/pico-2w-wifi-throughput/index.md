+++
title = "Tuning TCP transmit throughput on the Pico 2 W"
description = "Notes on getting the Raspberry Pi Pico 2 W's TCP transmit rate from 0.07 Mbit/s with the stock example to about 19.5 Mbit/s at stock clock, with the radio bus inside its spec."
date = 2026-10-17

[taxonomies]
tags = ["Rust", "embassy", "RP2350", "Wi-Fi", "firmware", "performance"]
+++

I'm building a battery-powered field recorder that offloads its SD card over Wi-Fi. The board is a Raspberry Pi Pico 2 W: an RP2350 plus an Infineon CYW43439 radio, which connects to the MCU over gSPI, a one-data-line SPI variant driven by a PIO program. Recordings are hundreds of megabytes, so I wanted to know how fast it can send over TCP.

The code is part of a larger private app, so there's no repo. These are the measurements and what I changed.

## Setup

All numbers are TCP with the Pico transmitting and a host receiving, counted at the host. Power is measured with an inline USB meter (a TC66) on the Pico's USB lead, so it covers the whole board.

The first measurements were on my home router's 2.4 GHz network, on a channel shared with four other APs. After that I used a dedicated AP: a USB Atheros AR9271 adapter on the host running a hotspot on channel 1, 802.11n HT20, with the Pico as the only client. The link sat at −57 dBm, MCS7 (72.2 Mbit/s PHY), with zero retries. When comparing configs I interleaved them sample by sample and compared medians.

## Results

| Step | Stack | CPU clock | gSPI bus | TCP Mbit/s |
|---|---|---|---|---|
| `pico-examples` iperf, as shipped | C SDK + lwIP | 150 MHz | 37.5 MHz | 0.07 |
| + power save off | lwIP | 150 | 37.5 | 0.6 |
| + larger lwIP heap and TCP windows | lwIP | 150 | 37.5 | 2–3 typical, 5.3 peak |
| Same, on the dedicated AP | lwIP | 150 | 37.5 | 3.7 median |
| CPU overclock | lwIP | 375 | 50 | 8.9 median |
| + out-of-spec bus, late-sample PIO program | lwIP | 375 | 75 | 12.7 median |
| PicoWi, hand-paced | PicoWi | 150 | 50 | ~13 |
| Rust: embassy `cyw43` + smoltcp | Rust | 150 | 25 | 15.3 median |
| + corrected PIO sampling program | Rust | 150 | 37.5 | ~19.5 median |

The first three rows are from the home network. The rest are from the dedicated AP.

## The stock example

The upstream `pico-examples` iperf in client mode managed 0.07 Mbit/s, with a ping RTT of 682 ms on a −48 dBm link. The SDK's default `CYW43_PERFORMANCE_PM` lets the radio sleep between beacons. With power save off, RTT dropped to 3–5 ms and throughput rose to 0.6 Mbit/s.

The rest was lwIP buffer sizes. Raising the heap from 4,000 to 32,000 bytes gave 2.2 Mbit/s, and raising the TCP windows from 8 to 32 MSS gave 3.1. A 128,000-byte heap with 40-MSS windows peaked at 5.3.

## A bug in my test sink

Results on the home network varied between 0.1 and 5.3 Mbit/s at constant signal strength. I assumed the channel was busy. The actual cause was my host sink: it served one connection at a time, and the iperf client opened a new connection every 10 seconds. When an old connection lingered half-open, the sink stayed blocked on it, the new connection filled the kernel buffer, the host advertised a zero window, and the Pico waited.

The power meter showed it. During a slow period the board drew about 340 mW, which is its idle draw, against about 1,000 mW when transmitting, and ping stayed at 1 ms with no retries. A congested channel would show retries, not an idle radio. A thread per connection in the sink fixed it.

## Overclocking with lwIP

On the dedicated AP the stock config was 3.7 Mbit/s median. The SDK's gSPI bus runs at `clk_sys / 4` and its driver exposes a fractional PIO divider, so CPU and bus speed can be changed independently.

With the bus fixed at 37.5 MHz, throughput rose with CPU clock up to about 300 MHz (5.3 Mbit/s) and then flattened. With the CPU at 375 MHz (the fastest that booted reliably, at a 1.30 V core), throughput followed the bus speed up to the CYW43439's 50 MHz limit: 8.9 Mbit/s median. Above 50 MHz the default sampling program connected but moved no data. The SDK's late-sample program, `spi_gap010_sample1`, worked at 62.5 and 75 MHz and gave 12.7 Mbit/s median (18.6 peak). At 93.75 MHz it wouldn't associate.

Power went up about 1.5× for about 3× the throughput, so energy per bit went from about 122 to about 55 mW per Mbit/s. But this needs a CPU well above its rated clock and a bus 50% over spec, which I don't want to ship.

## PicoWi

Next I tried PicoWi, Jeremy Bentham's from-scratch CYW43439 driver and minimal TCP/IP stack. Porting it to the RP2350 took a board define and two missing includes. It reached about 13 Mbit/s at 150 MHz with a 50 MHz bus, matching overclocked lwIP without the overclock, so a lot of lwIP's cost is software overhead.

PicoWi doesn't read the peer's receive window or the CYW43's bus credits, though. It sends a segment on every poll. Past about 13 Mbit/s it overruns the chip, drops segments, and falls to about 1 Mbit/s. It only worked with a hand-tuned token-bucket pacer that I added.

## embassy and smoltcp

embassy's `cyw43` driver gates every transmit on the chip's SDPCM bus credits, and smoltcp (under `embassy-net`) handles the TCP window and retransmission. I wrote a transmit-only test app: one socket, static IP, power save off, a 64 KB TX buffer, a 2 KB RX buffer, and `write_all` in a loop.

At a 25 MHz bus, the divider embassy's RP2350 Wi-Fi example uses, it got 15.3 Mbit/s median at stock clock with zero retries and no pacing. Some sweeps:

- TX buffer: 16 KB gave 11.5 Mbit/s, 32 KB gave 13.1, and 64 KB reached full rate. Larger gained nothing.
- The `firmware-logs` feature of cyw43 reads the radio's debug log on every poll. Turning it off was worth 4% (14.6 → 15.3).
- CPU clock: throughput scaled from 100 to 150 MHz, and 200 MHz added almost nothing. Energy per bit stayed at about 51–54 mW per Mbit/s from 100 to 150 MHz.

A faster bus didn't work. At 37.5 MHz the chip associated, but SDPCM reads came back corrupt ("len inv mismatch") and no data moved. 30 MHz stalled too.

## Finding the bottleneck

I guessed at the CPU, then the radio's airtime, then the bus, without measuring any of them. So I added an atomic counter to `cyw43-pio`'s transfer functions that totals every bit clocked over gSPI, and read it against the RP2350's DWT cycle counter. Bits divided by bus rate gives bus-active time, and that over wall-clock time gives bus utilization.

At 25 MHz, pushing about 18 Mbit/s of UDP, the bus was about 77% busy. The remaining 23% was CPU and the gaps between transactions. The radio wasn't the limit: one unaggregated 1500-byte frame at MCS7 takes about 368 µs including backoff, preamble and block-ack, which caps goodput at roughly 33 Mbit/s, so at 15–18 the radio was idle about half the time. It wasn't aggregating either. The CYW43439 rejects an A-MPDU limit of 16 (IOCTL error −29), and raising it from 4 to 8 changed nothing, because the host never had enough frames queued.

So the bus was the thing to speed up.

## The PIO sampling program

How fast the gSPI read path can run depends on where in the clock cycle the PIO program samples the input. The SDK's default, `spi_gap01_sample0`, idles on clock-high and samples on clock-low. The embassy default in my checkout did the reverse, and that was what corrupted reads above 25 MHz. I ported the SDK's program into `cyw43-pio`, and 37.5 MHz (an integer ÷2 of the stock clock, and inside the chip's spec) worked.

That gave about 19.5 Mbit/s median TCP at about 825 mW, or about 42 mW per Mbit/s, with zero retries and a stock 150 MHz CPU. Bus utilization fell to about 59%. 50 MHz needs a fractional ÷1.5 divider at stock clock, and that corrupted. Embassy's late-sample overclock program didn't work at any speed I tried on this board, even though it's instruction-for-instruction the same as the SDK's `spi_gap010_sample1`.

The chip still has to come up at 25 MHz. Bringing it up at 37.5 associates and completes the TCP handshake, but bulk data never flows, including after cold boots. So the firmware brings up at 25 MHz and switches the PIO divider to 37.5 after joining, the same way the pico-sdk's `cyw43_set_pio_clkdiv` works. The C SDK does bring up directly at high bus speeds, so this should be fixable, but I haven't fixed it.

UDP with the same bus fix measured 24.3 Mbit/s median at about 39 mW per Mbit/s. TCP is slower because every ACK has to come back over the same half-duplex wire. I'm keeping TCP for the offload because it's reliable.

## Upstream

The sampling fix was already upstream. embassy PR [#4792](https://github.com/embassy-rs/embassy/pull/4792), merged in October 2025, picks the gSPI PIO program by effective PIO clock and added a "high speed" program with the same nop-on-high, sample-on-low arrangement. My checkout was from August 2025. The bring-up divider switch and the bus-bit counter are local patches I haven't sent anywhere. Upstream has also fixed a DMA race in `cyw43-pio` that corrupted RP2350 reads during firmware load ([#6001](https://github.com/embassy-rs/embassy/pull/6001)). I haven't tested whether that explains the high-speed bring-up problem.

## Caveat

19.5 Mbit/s is the radio path fed from an in-memory source. Reading real files off the SD card through the full app is currently about 13 Mbit/s, and I haven't profiled that path yet.
