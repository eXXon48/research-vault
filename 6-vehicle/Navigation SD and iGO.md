---
tags: [cmu150, navigation, igo, nng, sd-card]
---

# Navigation SD and iGO

How the navigation SD card is detected and mounted, and where the map licensing decision is made.

## Overview

- `svcjcinavi.so` detects the internal SD slot by the USB ID of the built-in USB-SD bridge
  (`0424:4040`), mounts the card and checks the map database version.
- It contains no per-card authenticity check. The licensing decision comes from the NNG (iGO)
  navigation engine, and `svcjcinavi.so` only relays it.
- Any card in the internal slot passes detection. An external USB card reader has a different USB ID
  and is rejected.

Related: [[Custom Apps (HMI)]], [[SM Service Graph]], [[GPS (LDS)]].

## Reference

### Services

| SM service | Binary | Notes |
|---|---|---|
| `jcinavi` | `jci_service`, `/jci/navi/svcjcinavi.so` | `reset_board="yes"`; depends on `stage_navi`; launches and restarts `NNG` |
| `NNG` | `/data/navi/jci-linux_imx6_volans-release` | the iGO engine; `affinity_mask="0x02"` |

`stage_navi.sh` is an 11-byte stub (`#!/bin/sh`), linked from `bin` to `scripts`, with no logic.
The whole check is native, in `svcjcinavi.so` and the NNG engine.

### What `svcjcinavi.so` does

| Function | Behaviour |
|---|---|
| `isExpectedNaviDevice()` | walks the udev parents and compares `idVendor` and `idProduct` with fixed values. The only two 4-hex tokens in the binary are `0424` and `4040`: VID:PID `0424:4040` = SMSC/Microchip, the internal USB-SD bridge chip, not a card controller. The check identifies the internal slot, not the card. |
| `startNaviSDCheck()` | enumerates block devices through udev, picks the partition behind the bridge, calls `mountSDAndStartNavi()`. `thWatchMount()` does the same on hotplug (udev monitor). No card → `navi_PostEvent(4)`. |
| `mountSDAndStartNavi()` | mounts partition 1 at `/mnt/sd_nav` (vfat); if a second partition exists, mounts it at `/mnt/sd_nav2`. Otherwise logs "There is no 2nd partition on the SD-card, so … will not be mounted." |
| `ChangeLinkIfNeccessary()` | checks a file's content ("contents of the file is what it should be / different…", "%d bytes were read … instead of expected %d"). This selects the region variant (EU `/jci/nng/` or Taiwan `/jci/nng_twn/`); it is not a card check. |
| `getMapVersionNumber`, `NAVI_ComplyToDbVersion` | parse `/content/DbVersion.zip` with its own ZIP reader (`ReadEOCD`, `CDFH`, `LFH`): map database version compatibility |
| `NNG2NAVI_SDCardSeemsOK_Handler` | → `navi_PostEvent(5)` |
| `NNG2NAVI_SDCardProblem_Handler(…, problemType, restartInfo)` | posts error events 9, 10 or 11 and may call `executeSystemCommand(...)` |

The two handlers receive messages from the NNG engine (`/com/jci/navi2NNG`, `/com/NNG/Api/*`).

### Licensing decision

- The map content and license verdict (`SDCardSeemsOK` or `SDCardProblem` / `LICENSE_PROBLEM`) comes
  from the iGO engine (`/data/navi/jci-linux_imx6_volans-release`), which reads the map content and
  the `.lyc` files on the card.
- The `.lyc` file next to the engine,
  `/jci/nng/license_EU/Visteon_Mazda_UK_TMC_TrafficMaster.lyc`, is a TMC traffic license, not the map
  license.

## Verified on

Stock rootfs of firmware `cmu150_EU_74.00.324A` (static analysis).

## Open questions

- How the iGO engine validates the map content and `.lyc` files (for example, whether they are
  signed).
