+++
title = "rMVCI"
description = "A from-scratch Rust J2534 driver for the Toyota Mini-VCI cable, built by reverse-engineering the cable's firmware, plus open tooling to replace Techstream on Linux."
weight = 5

[taxonomies]
tags = ["Rust", "automotive", "reverse engineering", "firmware"]
+++

A Rust driver for the Mini-VCI diagnostic cable (native API, J2534 shared library, Android JNI, and a TCP bridge), with KWP2000 and UDS clients over K-line and CAN. Validated against 13 ECUs on a real Gen-2 Prius. Part of a larger effort to decode Techstream's vehicle database and rebuild its function in the open.

#### [Write-up](@/blog/rmvci/index.md) • [GitHub](https://github.com/FletcherD/rmvci) • [Release 0.1.0](https://github.com/FletcherD/rmvci/releases/tag/v0.1.0) {.centered-text}
