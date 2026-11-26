---
tags: [cmu150, hardware, firmware]
---

# Hardware and Firmware

The CMU150 hardware platform: SoC, CPU, I²C devices, board variants and the firmware it runs.

## Overview

The CMU150 is the Visteon/JCI generation of the Mazda Connect head unit, fitted to the
MX-5 ND Gen1. It is built on an i.MX6 Dual SoC with two Cortex-A9 cores at 996 MHz and runs
Linux 3.0.35 with a BusyBox userland. At start-up, a GPIO check decides between two board
variants (legacy and "WCP"), and each variant gets its own System Manager configuration.

Related: [[Wi-Fi and Bluetooth]], [[Firmware Update]], [[Partitions and Storage]],
[[Native Binary Support]].

## Unit

| | |
|---|---|
| Unit | Mazda Connect CMU150 (Visteon/JCI), MX-5 ND Gen1 |
| Kernel | Linux 3.0.35 |
| Userland | BusyBox v1.19.2 (built 2020-02-14), 274 applets ([[BusyBox]]); glibc 2.11.1 |
| RAM | ~764 MB total, ~230 MB free with the HMI running |
| OEM part number | `JCI_OEM_PART_NUMBER = NA1P-669C0-E` (`mfg-version.ini`, [[config-mfg Keys Partition]]) |
| Firmware | `cmu150_EU_74.00.324A` (EU region) |

## CPU

| | |
|---|---|
| SoC | i.MX6 Dual (`/proc/cpuinfo` revision `63000`, i.MX6Q/D die) |
| Cores | 2 × Cortex-A9, both online (`kernel_max` 1) |
| Clock | 996 MHz; available steps `996000 792000 396000` kHz |
| cpufreq governor | `userspace`, both cores at `996000` kHz |
| Thermal | zone `anatoptz`, cpufreq as cooling device; trip points 80 / 90 / 105 °C; ~31 °C on an idle bench unit |

- 996 MHz is the fused limit. `/sys/fsl_otp/HW_OCOTP_CFG3` = `0x620302`: SPEED_GRADING bits
  [17:16] = `0b10` is the 996 MHz grade (`0b11` would be 1.2 GHz).
- The part has two CPUs in hardware. It is not a Quad with cores turned off: the Cortex-A9
  SCU configuration register (`devmem 0x00A00004` = `0x00005531`, bits [1:0] = `01`)
  reports two CPUs.
- The machine name `Hardware: JCI i.MX 6Quad Core CMU Board` in `/proc/cpuinfo` is only a
  name string.

## I²C devices

| Bus-addr | `name` | Kernel driver | Function |
|---|---|---|---|
| `0-0018` | `aic310x` | `aic310x` | audio codec |
| `1-0020` | `adv7180` | `adv7180` | video decoder (camera input) |
| `2-0010` | `auth-ic` | none | Apple Authentication Co-Processor (MFi) |

Stock i2c-tools: `/usr/sbin/i2cdetect`, `i2cdump`, `i2cget`, `i2cset`.

### Apple authentication co-processor

- Bus `i2c-2` (`imx-i2c`); node `/dev/i2c-2`, `crw------- cmu root 89,2`.
- No kernel driver binds `auth-ic`. Userspace talks to it through `/dev/i2c-2`
  (`I2C_SLAVE_FORCE`) and opens the node per transaction, not for the whole run.
- Users of the chip:
  - `/usr/bin/mfid`: SM service `mfid`, `autorun="yes"` in `jci/sm/sm.conf` and
    `jci/sm/sm_WCP.conf`. It runs alongside `/usr/bin/carplayd` and SM service `jciCARPLAY`.
  - `/usr/bin/ipoddev_full_auth`.
  - `jci/lib/libjcii2ccp.so`.
- `/usr/bin/auth_ic_test` is the stock diagnostic for the chip:

  | Option | Meaning |
  |---|---|
  | `-d <dev>` | device, default `/dev/i2c-2` |
  | `-f` | firmware version |
  | `-p` | protocol version |
  | `-r` | reset (writes the GPIO port directly through `/dev/mem`) |
  | `-q` | quiet |
  | `-w <ms>` | delay between messages |

  Registers it knows: firmware/protocol version, device ID, error code, control/status,
  signature, challenge, certificate data 1–15, self-test.

## Board variants and SM configuration

`/usr/bin/autostart` (shell script) starts the System Manager with one of two configurations.
The exit code of `/jci/scripts/get_board_type.sh` (also linked as `/jci/bin/get_board_type.sh`)
picks the configuration:

| GPIOs `DI_TP1056`, `DI_TP1029` | Exit code | Board | SM configuration |
|---|---|---|---|
| both `0`, or GPIOs absent | `1` | legacy | `SMCFG_NORMALMODE=/jci/sm/sm.conf` |
| either one `1` | `2` ("For WCP, New Hardware") | WCP | `SMCFG_WCPMODE=/jci/sm/sm_WCP.conf` |

The script reads `/sys/class/gpio/DI_TP1056/value` and `/sys/class/gpio/DI_TP1029/value`,
with 3 retries 1 s apart. On a legacy board both read `0` and SM runs `-f /jci/sm/sm.conf`.

The two configurations differ in 13 places. The WCP configuration targets different hardware
(u-blox GPS, `new_hw` for Android Auto):

| Service | `sm.conf` | `sm_WCP.conf` |
|---|---|---|
| `jciMMUI` args | `…display=filtered-touchscreen0` | `…, wcp` |
| `start_network`, `start_wifi` args | `UCP` | empty |
| `jciUSBMGR`, `jciCARPLAY`, `jciBCA`, `jcinavi`, `blm_nms_svc` args | none | `wcp` |
| `jciAAPA` args | none | `new_hw` |
| `bds` args | `-interface dbus` | `-interface dbus -wcp` |
| `jciLDS` args | `…lds.xml` | `…lds.xml GPS=UBLOX`, plus dependency `stage_navi` |
| one service | — | extra dependency `emmc_mount` |

## Firmware

- `cmu150_EU_74.00.324A` is compatible with the MX-5 ND Gen1: the CX-5 and the MX-5 share the
  CMU150 platform and the EU 74.xx line.
- Upgrade path: 74.00.311 → 74.00.324.
- On firmware 74, password login for the `cmu` shell account is disabled by default. The
  old `jci` default password no longer matches, and access is key-based
  ([[config-mfg Keys Partition]], [[SSH Access]]).
- Update package, update steps and what each partition keeps: [[Firmware Update]].

## How to: write to the root filesystem

The root filesystem is relfs, mounted read-only ([[Partitions and Storage]]).

1. Optional: pause the hardware watchdog: write `1` to `/sys/class/gpio/Watchdog Disable/value`.
2. `mount -o rw,remount /` ([[mount]]).
3. Make the change.
4. `mount -o ro,remount /`.
5. If you paused the watchdog, write `0` back to `/sys/class/gpio/Watchdog Disable/value`.

A firmware update rebuilds the root filesystem, so every root-filesystem change must be
applied again after an OS update.

## Verified on

Firmware `cmu150_EU_74.00.324A`, legacy board.
