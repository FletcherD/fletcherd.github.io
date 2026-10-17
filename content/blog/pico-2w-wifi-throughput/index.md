+++
title = "Pico 2 W Wi-Fi: from 0.07 to 19.5 Mbit/s"
description = "How I took the Raspberry Pi Pico 2 W's TCP transmit throughput from 0.07 Mbit/s with the stock example to about 19.5 Mbit/s at stock clock and an in-spec bus, by measuring each bottleneck instead of guessing at it."
date = 2026-10-17

[taxonomies]
tags = ["Rust", "embassy", "RP2350", "Wi-Fi", "firmware", "performance"]
+++

*How I took the Pico 2 W's Wi-Fi transmit rate from 0.07 Mbit/s with the stock example to about 19.5 Mbit/s, at stock clock and with the radio bus inside its spec, mostly by measuring things I had been guessing about.*

---

I'm building a battery-powered field recorder that offloads its SD card over Wi-Fi. The board is a Raspberry Pi Pico 2 W: an RP2350 plus an Infineon CYW43439 radio, which hangs off the MCU on a one-data-line SPI variant called gSPI, driven by a PIO program. Recordings are hundreds of megabytes, so the question was simple: how fast can this thing push bytes out over TCP?

The work took four days, July 29 to August 1, 2026. The code lives inside a larger private app, so there's no repo; this post is the method and the numbers.

## The setup

Every number below is **TCP, Pico transmitting, host receiving**, which is the offload direction. Throughput is counted at the host sink, not on the device. Power is measured with an inline USB meter (a TC66) on the Pico's USB lead, so it covers the whole board.

The first measurements ran on my home router's 2.4 GHz network, on a channel shared with four other APs, at −48 to −51 dBm. After that I moved to a dedicated AP: a USB Atheros AR9271 adapter on the host running a hotspot on channel 1, 802.11n HT20. The Pico was the only client on that network, and the link held a steady −57 dBm at MCS7 (72.2 Mbit/s PHY) with zero retries. When comparing configs I interleaved them sample by sample, so they all saw the same drifting channel, and compared medians.

## The progression

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
| + corrected PIO sampling program | Rust | 150 | 37.5 | **~19.5 median** |

The first three rows are from the home network. The rest are from the dedicated AP.

## Baseline: 0.07 Mbit/s, and a 682 ms ping

I started with the upstream `pico-examples` iperf in client mode, flashed over SWD, plus a host TCP sink counting bytes. It managed 0.07 Mbit/s. Ping RTT was 682 ms on a −48 dBm link, which pointed straight at power save: the SDK's default `CYW43_PERFORMANCE_PM` lets the radio sleep between beacons. Turning power save off brought the RTT down to 3–5 ms and the throughput up to 0.6 Mbit/s.

The rest of that first pass was lwIP tuning. Raising the heap from 4,000 to 32,000 bytes got 2.2 Mbit/s. Raising the TCP windows from 8 to 32 MSS got 3.1. A 128,000-byte heap with 40-MSS windows peaked at 5.3. My takeaway from this stage: a Pico W throughput figure means nothing unless it comes with its power-save mode and lwIP buffer sizes.

## A fake collapse

Results on the home network jumped around between 0.1 and 5.3 Mbit/s at constant signal strength. I put that down to a busy channel. That was wrong. The real cause was my host sink: it served one connection at a time, and the iperf client opened a new connection every 10 seconds. When an old connection lingered half-open, the sink stayed blocked on it. The new connection filled the kernel buffer, the host advertised a zero window, and the Pico politely waited.

The power meter gave it away. During a "collapse" the board drew about 340 mW, its idle draw, compared with about 1,000 mW when it was really transmitting, and ping stayed at 1 ms with no retries. A crowded channel gives you retries and wasted airtime. It doesn't give you an idle radio on a 1 ms link. Once the sink used a thread per connection, the collapses went away. The general rule: if throughput collapses while the radio sits at idle current, look at the sender and receiver before blaming the air.

## Overclocking lwIP: faster, but only by leaving the spec

On the dedicated AP the stock config was 3.7 Mbit/s median. The SDK's gSPI bus runs at `clk_sys / 4`, and its driver exposes a fractional PIO divider, so I could change CPU speed and bus speed independently.

With the bus fixed at 37.5 MHz, throughput rose with CPU clock up to about 300 MHz (5.3 Mbit/s) and then stopped rising. With the CPU at 375 MHz (the fastest that booted reliably, at a 1.30 V core), throughput tracked the bus speed up to the CYW43439's 50 MHz spec limit: 8.9 Mbit/s median. Above 50 MHz, the default sampling program connected but moved no data. The SDK ships several gSPI PIO programs that sample the returning data at different points, and the late-sample `spi_gap010_sample1` worked at 62.5 and 75 MHz. That gave 12.7 Mbit/s median (18.6 peak), with a 375 MHz CPU and a bus running 50% over spec. At 93.75 MHz it wouldn't associate at all.

Power went up about 1.5× for about 3× the throughput, so energy per bit dropped from about 122 to about 55 mW per Mbit/s. But it depends on a CPU far above its rated clock and a bus well beyond spec, which I didn't want in a shipping device.

## Dead end: skipping flow control

Next I tried PicoWi, Jeremy Bentham's from-scratch CYW43439 driver and minimal TCP/IP stack. Porting it to the RP2350 took a board define and two missing includes. It reached about 13 Mbit/s at stock 150 MHz with an in-spec 50 MHz bus, which matched the overclocked lwIP result with none of the overclocking. That told me a lot of lwIP's cost was software overhead.

The catch was that PicoWi never reads the peer's receive window or the CYW43's bus credits. It sends a segment on every poll. Past about 13 Mbit/s it overruns the chip, drops segments, and its recovery falls apart down to about 1 Mbit/s. It only worked because I added a hand-tuned token-bucket pacer.

What I wanted was PicoWi's lean data path with proper flow control.

## Rust and embassy: 15.3 Mbit/s with no pacing

That combination already exists. embassy's `cyw43` driver gates every transmit on the chip's SDPCM bus credits, and smoltcp underneath `embassy-net` handles the TCP window and retransmission. Both are `no_std` and allocation-free. I wrote a minimal transmit-only app with one socket, a static IP, power save off, a 64 KB TX buffer and a 2 KB RX buffer (it only ever receives ACKs), and called `write_all` in a loop.

Bringing up at 25 MHz, the conservative divider embassy's RP2350 Wi-Fi example uses, it got 15.3 Mbit/s median at stock clock with zero retries and no pacing. Sweeps showed:

- **TX buffer:** 16 KB gave 11.5 Mbit/s, 32 KB gave 13.1, and 64 KB reached full rate. Going bigger gained nothing.
- **`firmware-logs` off:** this cyw43 feature reads the radio's debug log on every poll. Disabling it was a free 4% (14.6 → 15.3).
- **CPU clock:** 100 → 150 MHz scaled throughput, and 200 MHz added almost nothing. Energy per bit stayed flat at about 51–54 mW per Mbit/s from 100 to 150 MHz.

Trying a faster bus failed. At 37.5 MHz the chip associated, but SDPCM reads came back corrupt ("len inv mismatch") and no data moved. 30 MHz stalled too.

## Measuring the bottleneck

Over the next hour I blamed the CPU, then the radio's airtime, then the bus. All three were explanations I had reasoned my way to, not ones I had measured. So I added a measurement. I put an atomic counter in `cyw43-pio`'s transfer functions that adds up every bit clocked over gSPI (command, data and status words), and read it against the RP2350's DWT cycle counter. Bits divided by bus rate gives bus-active time, and dividing that by wall-clock time gives bus utilization.

At 25 MHz, while pushing about 18 Mbit/s of UDP, **the bus was about 77% busy**. The remaining ~23% was CPU and the latency between transactions. A bit of airtime arithmetic rules out the radio: a single, unaggregated 1500-byte frame at MCS7 takes about 368 µs including backoff, preamble and block-ack. That caps goodput at roughly 33 Mbit/s, so at 15–18 the radio was idle about half the time. It wasn't aggregating either. The CYW43439 rejects an A-MPDU limit of 16 (IOCTL error −29), and raising it from 4 to 8 changed nothing, because the host never kept enough frames queued for aggregation to kick in.

So the radio and CPU had headroom, and the thing to speed up was the bus.

## The sampling program

The gSPI read path is a PIO program, and where in the clock cycle it samples the input decides how fast it can go. Comparing the embassy program in my checkout with the pico-sdk's showed the difference. The SDK's default, `spi_gap01_sample0`, idles on clock-high and samples on clock-low. The embassy default I had did the reverse, and that was what corrupted reads above 25 MHz. I ported the SDK's program into `cyw43-pio`, and 37.5 MHz, an integer ÷2 of the stock clock and inside the chip's spec, started working.

**Result: about 19.5 Mbit/s median TCP, about 825 mW, about 42 mW per Mbit/s, zero retries, stock 150 MHz CPU.** Bus utilization fell to about 59%. 50 MHz needs a fractional ÷1.5 divider at stock clock, and that one corrupted. Embassy's late-sample overclock program didn't work at any speed I tried on this board, even though it's instruction-for-instruction the SDK's `spi_gap010_sample1`.

One problem remained. The chip still has to **come up at 25 MHz**. Bringing it up directly at 37.5 associates and even completes the TCP handshake, but bulk data never flows (that held across cold boots). So the firmware brings up at 25 MHz and switches the PIO divider to 37.5 after joining, the same way the pico-sdk's `cyw43_set_pio_clkdiv` works. The C SDK does bring up directly at high bus speeds, so this is fixable. I haven't fixed it yet.

For comparison, UDP with the same bus fix measured 24.3 Mbit/s median at about 39 mW per Mbit/s. That's because UDP removes the TCP ACK stream, and every ACK has to come back across the same half-duplex wire. I kept TCP for the offload anyway, because it's reliable.

## Upstream status

I didn't submit the sampling-program fix, and it turns out it didn't need submitting. While writing this post I checked upstream embassy: PR [#4792](https://github.com/embassy-rs/embassy/pull/4792), merged in October 2025, already picks the gSPI PIO program by effective PIO clock. It added a "high speed" program, the same nop-on-high / sample-on-low arrangement, for exactly this case: a stock RP2350 with the default ÷2 divider. My embassy checkout dated from August 2025, so I rediscovered a bug that had already been fixed. The bring-up divider switch and the bus-bit counter are local patches I haven't sent anywhere. Upstream has also since fixed a DMA race in `cyw43-pio` that corrupted RP2350 reads during firmware load ([#6001](https://github.com/embassy-rs/embassy/pull/6001)). I haven't tested whether that's what makes high-speed bring-up fragile here.

## What I'd keep

- Count bytes at the receiver, and read the power meter. An idle-current "collapse" is a stalled sender or receiver, not RF.
- Interleave the configs you're comparing, and compare medians.
- When the explanations keep changing, stop and instrument. A counter and a cycle timer settled what hours of reasoning hadn't.
- On this chip, flow control isn't optional. Credit-gated transmit plus a real TCP window beat every hand-paced or overclocked setup I tried, and used less energy per bit doing it (about 42 versus about 55 mW per Mbit/s).

One more caveat: 19.5 Mbit/s is the radio path fed from a synthetic in-memory source. Pulling real files off the SD card through the integrated app is currently about 13 Mbit/s, and that path still needs its own round of measuring.
