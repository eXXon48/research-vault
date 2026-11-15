---
tags: [cmu150, navigation, igo, nng, sd-card]
---

# Navigation SD and iGO

Where the navigation-SD authenticity and licensing gate lives in firmware `74.00.324A`
(`svcjcinavi.so` and the NNG/iGO engine).

## Stack
- `sm.conf`: `jcinavi` (`jci_service`, `/jci/navi/svcjcinavi.so`, **`reset_board=yes`**)
  → depends on `stage_navi` → launches `NNG` = `/data/navi/jci-linux_imx6_volans-release`
  (the iGO engine, `affinity_mask=0x02`, restarted by jcinavi).
- `stage_navi.sh` is an **11-byte stub** (`#!/bin/sh`), symlink bin→scripts. No logic.
  The whole gate is native, in `svcjcinavi.so` + the NNG engine.

## What `svcjcinavi.so` does (plumbing and relay, not authentication)
1. **Device detection — `isExpectedNaviDevice()`**: walks udev parents, does
   `strcmp(idVendor, EXPECTED)` && `strcmp(idProduct, EXPECTED)`. The only two
   4-hex tokens in the binary are **`0424`** and **`4040`** → VID:PID
   **`0424:4040` = SMSC/Microchip**, the head unit's **internal USB-SD bridge
   chip**, not a card controller. Any card in the built-in slot therefore passes the
   VID/PID check: it identifies the internal slot, not a genuine card. An external USB card
   reader has a different VID/PID and is rejected; only the internal slot passes.
2. **`startNaviSDCheck()`**: udev-enumerates block devices, picks the `partition`
   behind the bridge, calls `mountSDAndStartNavi()`. `thWatchMount()` does the same
   on hotplug (udev monitor). No card inserted → `navi_PostEvent(4)`.
3. **`mountSDAndStartNavi()`**: mounts partition **1** at `/mnt/sd_nav` (vfat), and
   if a **2nd partition** exists, mounts it at `/mnt/sd_nav2`. (Hence the log
   "There is no 2nd partition on the SD-card, so … will not be mounted.")
4. **`ChangeLinkIfNeccessary()`**: the "contents of the file is what it should be /
   different…" + "%d bytes were read … instead of expected %d" check. This is
   **region/variant link selection** (EU `/jci/nng/` vs Taiwan `/jci/nng_twn/`),
   not card authenticity.
5. **`getMapVersionNumber` / `NAVI_ComplyToDbVersion`**: parse `/content/DbVersion.zip`
   (own ZIP reader: ReadEOCD/CDFH/LFH) → map **DB-version compatibility** check.
6. **Relay only:** `NNG2NAVI_SDCardSeemsOK_Handler` → `navi_PostEvent(5)`;
   `NNG2NAVI_SDCardProblem_Handler(…, problemType, restartInfo)` → posts error
   events 9/10/11 and may `executeSystemCommand(...)`. These are handlers for
   messages the NNG engine sends (`/com/jci/navi2NNG`, `/com/NNG/Api/*`).

## Where the licensing verdict lives
`svcjcinavi.so` contains **no per-card crypto/authenticity check**. It mounts whatever vfat
is in the internal slot and asks NNG. The map-content / license verdict (`SDCardSeemsOK` vs
`SDCardProblem` / `LICENSE_PROBLEM`) is produced by the iGO engine
(`/data/navi/jci-linux_imx6_volans-release`), reading the licensed (likely signed) map content
and `.lyc` files on the card. The `.lyc` beside the engine
(`/jci/nng/license_EU/Visteon_Mazda_UK_TMC_TrafficMaster.lyc`) is a **TMC traffic** license,
not the map master.

See also [[Custom Apps (HMI)]].
