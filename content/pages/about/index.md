+++
title = "About"
template = "info-page.html"
path = "about"
+++

I'm Fletcher, an independent embedded systems engineer based in Southern California, with an electrical engineering degree from Georgia Tech and experience going back to 2011. I bring up unfamiliar boards and turn them into full-featured systems, whether that means embedded Linux, an RTOS, or bare metal. I've shipped firmware and software for medical devices, cars, satellites, access control equipment, and robotics.

I'm available for contract work, remote or on-site in SoCal. See [services](@/pages/services/index.md) for what I can help with.

## Experience

**Senior Firmware Engineer**, Camino Robotics (May 2026 – present)
- Early employee at a hardware startup building a smart walker for people with disabilities; responsible for all device firmware.
- Solved EMC compliance issues under pressure, saving runway.
- Designed control algorithms for power assist, slip detection, and user safety, and built the complete OTA update feature, including cloud and DFU bootloader components.

**Senior Embedded Software Engineer**, Amazon (Oct 2025 – Mar 2026)
- Built flight software components for Amazon Kuiper, Amazon's satellite product, including a Rust system monitor service that watches hardware sensors and reports faults.
- Integrated I2C voltage and temperature sensors and peripheral drivers into userspace.
- Created a CI pipeline to deploy firmware and run hardware-in-the-loop tests.

**Lead Firmware Engineer**, Mercury Systems (Oct 2024 – Mar 2025)
- Brought up hardware for an all-new line of access control equipment, including an STM32-based embedded Linux image built with Yocto, Ethernet, eMMC, and PMIC.
- Implemented secure boot across all four bootloader stages.
- Wrote a Rust test server for testing hardware over an API, plus a GitLab/Jenkins CI pipeline.

**Senior Embedded Software Engineer (contract)**, Polaris Industries (Oct 2023 – Mar 2024)
- Built features for an automotive dashboard display in C++ and Qt, with backend connections to the vehicle ECU over CAN.

**Senior Embedded Software Engineer**, Lordstown Motors (Oct 2022 – Mar 2023)
- Led development of secure OTA updates for a car ECU on ARM Cortex-A8: a bare-metal secure bootloader in Rust, and a FreeRTOS network task in C that receives the image over TCP, verifies it, and flashes it.
- Implemented secure boot and firmware image signing; interfaced with other vehicle components over CAN and automotive Ethernet.
- Wrote a desktop firmware update application in Rust.

**Algorithm Engineer (contract)**, Edwards Lifesciences (May 2021 – Oct 2022)
- Developed real-time sensor fusion algorithms for medical devices, with Python tests to characterize them and visualize quality.

**Algorithm Engineer**, InBrace (Oct 2019 – Mar 2020)
- Automated the creation of a dental brace from a 3D dental scan: Python geometry processing that turns a volumetric scan into 3D-printable meshes, with OpenGL visualizations of quality.

**Software Engineer (remote contract)**, DePuy Synthes (Jan 2019 – Jun 2019)
- Developed C++ extensions to CAD software for surgery planning.

**Software Engineer (contract)**, Comtech Telecommunications (Jun 2018 – Dec 2018)
- Developed embedded mapping software for car infotainment systems in C++ and Qt on embedded Linux (i.MX6).

**Software Engineer**, Promenade Software (Jan 2016 – Nov 2017)
- Developed C++ software and a Qt/QML touchscreen interface for a medical device running embedded Linux on ARM Cortex.
- Integrated a BLE module over SPI, I2C, and USB, including kernel drivers, and designed data export from the device to a phone app over BLE or Wi-Fi.

**Software Engineer (contract)**, Lantos Technologies (Jan 2016 – May 2018)
- Developed a Windows application in C++14, Qt, OpenGL, and custom GLSL shaders for displaying and editing 3D scan data.

**Embedded Software Engineer**, Lantos Technologies (Jun 2013 – Oct 2015)
- Embedded C for a 3D imaging medical device, including image processing and network transfer on bare metal.
- Designed QA and test procedures for hardware and software, and worked across hardware and software under FDA regulations.

**Software Engineer**, Charles River Analytics (Jun 2011 – Jun 2013)
- Turned algorithms from scientists and white papers into optimized C++, for example detecting heart rate from standard webcam video.

## Personal projects

- **Custom Linux firmware for the Grom VLine VL2**, an aftermarket car computer: patched kernel, U-Boot, and a new device tree for its graphics chip and audio codec, plus a bare-metal Rust driver for the vehicle's IEBus/AVC-LAN bus.
- **Meldonica**, a musical instrument played with a soft silicone controller: analog circuitry to measure deformation, BLE to the host, and Rust firmware and a MIDI frontend.
- **[Tank War 2000](@/blog/tank-war-2000/index.md)**, a multiplayer web game built in two weeks for the Vibe Coding Game Jam.

## Education

**B.S. Electrical Engineering**, Georgia Institute of Technology, 2011, graduated with high honors.

Undergraduate research included a charge-integrating electrometer for measuring very small currents from ionization, the telecommunications protocol for the university's nanosatellite program, and a computer vision lane detection SDK.

## Contact

Find me on [GitHub](https://github.com/FletcherD).
