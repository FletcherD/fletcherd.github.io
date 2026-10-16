+++
title = "TC66C custom firmware"
description = "From-scratch Rust firmware for a $30 USB-C power meter that triggers USB-PD, PPS, QuickCharge, and Samsung AFC on command."
weight = 10

[taxonomies]
tags = ["Rust", "firmware", "reverse engineering", "USB-PD"]
+++

A clean-room replacement firmware for the RuiDeng TC66C USB-C power meter, written in `no_std` Rust for its GD32F150 microcontroller. It self-flashes over USB, emulates a ChargerLAB POWER-Z KM003C so the mainline Linux `powerz` driver binds to it, and triggers fast-charge protocols on command from a PC.

#### [Write-up](@/blog/tc66-custom-firmware/index.md) • [GitHub](https://github.com/FletcherD/tc66-custom-fw) {.centered-text}
