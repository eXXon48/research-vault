---
tags: [cmu150, mazda, moc]
---

# CMU150 research

Reference documentation for the Mazda Connect **CMU150** head unit (Visteon/JCI), as fitted to
the **MX-5 ND Gen1**, firmware **cmu150_EU_74.00.324A**. It covers the hardware, storage, boot,
system services, the HMI and the vehicle interfaces.

The files form an [Obsidian](https://obsidian.md) vault. Every file is plain Markdown and reads
fine on GitHub. Part of [cmu-REvised](https://github.com/eXXon48/cmu-REvised).

## Platform at a glance

| | |
|---|---|
| SoC | i.MX6 Dual, 2 × Cortex-A9 at 996 MHz, ~764 MB RAM |
| Kernel | Linux 3.0.35 |
| Root filesystem | transactional relfs, mounted read-only |
| Writable storage | `/data` (NAND), `/data_persist` (eMMC) |
| HMI | Opera 12 (Presto) web application under `/jci/gui` |
| Service manager | JCI System Manager (SM), `/jci/sm/sm.conf` |
| Root access | `cmu` is uid 0; key-only SSH ([[SSH Access]]) |

## Contents

### 1. Hardware and storage (`1-hardware/`)
- [[Hardware and Firmware]]: SoC, CPU, I²C devices, board variants, root filesystem writes
- [[Wi-Fi and Bluetooth]]: the WiLink combo chip, 2.4 GHz only, coexistence
- [[Firmware Update]]: the `.up` package, its steps, what each partition keeps
- [[Partitions and Storage]]: MTD layout, mounts, writable surfaces, what survives an update
- [[config-mfg Keys Partition]]: the read-only keys and calibration squashfs
- [[Native Binary Support]]: executable format, ABI, toolchain notes
- [[FlashFX Read Path]]: how the root filesystem is read from NAND and what it costs

### 2. Boot (`2-boot/`)
- [[Boot Chain and Fail-Safe]]: SPI-NOR layout, the `boot-select` flag, the fail-safe image
- [[Boot Timeline]]: when each SM stage and service starts in a normal boot
- [[Boot Speed Levers]]: changes that move boot milestones, with measured effect
- [[System Clock]]: unset system clock, hardware RTC, time service

### 3. System services (`3-system/`)
- [[SM Service Graph]]: services, their dependencies and IPC
- [[SM Service Control (smctl)]]: controlling SM services from a shell
- [[Users and Privileges]]: accounts, file ownership, process uids, device and D-Bus permissions
- [[Syslog-ng]]: logging configuration and log locations
- [[BusyBox]]: the BusyBox applets on the unit, one page per command in `commands/`

### 4. HMI (`4-hmi/`)
- [[4-hmi/jci-ui/README|HMI framework reference]] (`jci-ui/`): Opera stack, messaging bus, framework API, app anatomy
- [[Custom Apps (HMI)]]: HMI extension points, sockets, vehicle-data path
- [[HMI Performance Levers]]: CPU, SM, Opera and framework settings that affect HMI speed
- [[Touch Speed Restriction]]: how touch input is disabled while driving
- [[Screen Capture]]: `/usr/bin/screenshot` and video recording

### 5. Audio and display (`5-audio-video/`)
- [[Audio Architecture]]: audio stack, ALSA devices, Audio Manager arbitration
- [[Audio Source Registration]]: Audio Manager D-Bus surface, stream types, mux PCMs
- [[Display Architecture]]: Wayland/IVI-shell stack, native clients, factory test mode
- [[Media Album Art]]: album art sources for USB and Bluetooth media, Gracenote

### 6. Vehicle interfaces (`6-vehicle/`)
- [[Vehicle Data (VBS)]]: VBS modules, D-Bus names, vehicle signals, what reaches the HMI
- [[Vehicle VIN]]: the D-Bus VIN method and model decoding
- [[GPS (LDS)]]: Location Data Service, GPS serial port, receiver selection
- [[Navigation SD and iGO]]: navigation SD detection and licensing

### 7. Access (`7-access/`)
- [[SSH Access]]: sshd configuration, accounts, boot-time behaviour, firewall rules

## Document format

Each document starts with one line on what it covers, then:

1. **Overview**: how the part works, in a few sentences.
2. **Reference**: paths, tables, commands and configuration.
3. **How to**: step-by-step procedures, where there are any.
4. **Measurements**: timings and counts, with the conditions they were taken under.
5. **Verified on**: firmware and board the facts were checked against.
6. **Open questions**: what is not known yet.

Sections that would be empty are left out.

## Scope

Observations and descriptions produced by reverse engineering, for interoperability and repair.
No firmware, binaries, keys or map data are included; see [`NOTICE.md`](NOTICE.md).
