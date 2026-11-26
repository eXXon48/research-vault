---
tags: [cmu150, hardware, wifi, bluetooth]
---

# Wi-Fi and Bluetooth

The CMU150 radio: one TI WiLink combo chip and one antenna, 2.4 GHz only. Covers driver,
client mode, coexistence and why 5 GHz is unavailable.

## Overview

Wi-Fi and Bluetooth share one TI WiLink `wl12xx` chip on a shielded module, connected over
SDIO, and one printed antenna. The FCC grant covers 2.4 GHz only. The driver declares a
5 GHz band, but that band has no channels: the factory calibration file selects a
single-band front-end profile. When Bluetooth pages a phone that does not answer, Wi-Fi
scans come back empty, because both radios share the chip and the antenna.

## Reference

### Regulatory and hardware

- FCC ID `CB262932` (Johnson Controls Interiors, label `MAZDA_GEN_65_CMU`, granted 2013-07-16).
  The grants cover 2.4 GHz only: Bluetooth 2402–2480 MHz, WLAN 2412–2462 MHz. There is no
  5 GHz (UNII) grant.
- Internal photos (fccid.io exhibit 2017138) show one radio module and one antenna:
  - The module is a TI WiLink combo chip on a castellated module under a shield can. The
    photos also show the bare module and the can removed.
  - The antenna is a printed L-shaped trace on the board edge.
  - The module carries one small front-end IC. This is consistent with the single-band NVS
    profile below.

### Driver and firmware

| | |
|---|---|
| Chip | TI WiLink `wl12xx`, SDIO (`SDIO_ID=0097:4076`) |
| Driver | `wl1271_sdio`, backport `ol_R5.SP8.01` |
| Firmware | Rev 7.3.10.0.142 |
| BT codec link | `wl1285q-bt` |
| Tools | stock `/usr/sbin/iw`, `/usr/sbin/hostapd`; interface modes include `AP` |
| Client mode | `wpa_supplicant -c /data/nms/wpa_supplicant-wlan0.conf -i wlan0 -D nl80211` |

### Bands

| | |
|---|---|
| Band 1 (2.4 GHz) | channels 1–14 |
| Band 2 (5 GHz) | declared (HT20, MCS 0–7), **no frequencies** |
| Regulatory domain at runtime | `country 00` (world); its 5 GHz ranges are `PASSIVE-SCAN, NO-IBSS` |

### Why the 5 GHz band is empty

The NVS calibration file has `dual_mode_select = 0`. The wl12xx driver turns on 11a only when
that flag is set. Otherwise the 5 GHz band keeps zero channels.

- File chain: `/lib/firmware/ti-connectivity/wl1271-nvs.bin` → `/var/run/wl1271-nvs.bin` →
  `/config-mfg/wl1271-nvs.bin` (1113 bytes, mode 0600, factory partition).
- `/jci/scripts/check-config.sh` creates the file only when it does not exist:
  `calibrator plt autocalibrate wlan0 <wl12xx_sdio.ko> /usr/share/ti/wifi-utils/ini_files/128x/TQS_S_2.6.ini …`.
- `TQS_S_2.6.ini` is a TriQuint **single-band** front-end profile:
  `Single_Dual_Band_Solution = 00`, `FemVendorAndOptions = 01` (TQS), `XTALItrimVal = 04`,
  `Platform_configuration = 02`. Dual-band profiles ship alongside it but are unused
  (`TQS_D_1.0.ini`, `TQS_D_1.7.ini`: `Single_Dual_Band_Solution = 01`).
- In the factory NVS, the general parameters start at byte offset 468:
  `xtal_itrim 0x04, platform_conf 0x02, dual_mode_select 0x00`. `dual_mode_select` is byte 477.
- `wl12xx.ko` strings: the NVS request `ti-connectivity/wl1271-nvs.bin`, and the debug line
  `11a is %ssupported` after each firmware boot. In the upstream wl12xx driver of this era:
  - the driver fetches the NVS once, when the hardware registers;
  - with 11a off, firmware boot sets the 5 GHz band's channel count to 0.

  So a changed NVS takes effect only after a module reload or a reboot.

### Boot auto-connect

- About 20 s after `NMS` starts, `blm_nms` (log tag `Generic`) enters `WiFiAutoConnecting`.
- While no saved network shows up in the scan, `wpa_supplicant` rescans every ~5 s:
  `CTRL-EVENT-SCAN-STARTED` → `SCAN-RESULTS` → `NETWORK-NOT-FOUND`.
- The join time therefore depends on when a scan first contains the access point, not on the
  boot.

### Wi-Fi/Bluetooth coexistence

Bluetooth pages a paired phone for HFP: `HFP connect start` → `Handsfree: Disconnected->Connecting`.
If the phone does not answer, `Page time out is occurred` follows after 5 s, and the next try
starts ~5 s later. This repeats while the phone stays silent. During paging, Wi-Fi scans
return no BSS at all, even with the access point 2 m away. The first sighting of the access
point comes in a gap between page attempts.

### Weak-link symptoms in the log

- scans that list no BSS at all;
- `SME: Trying to authenticate` without `Associated`;
- kernel `wl12xx: WARNING Unable to flush all TX buffers, timed out (timeout 500 ms`;
- `CTRL-EVENT-DISCONNECTED ... reason=3 locally_generated=1` (the station leaves).

## Measurements

| Condition | Result |
|---|---|
| Phone answers the HFP page in 2 s | Wi-Fi connected 8 s after `WiFiAutoConnecting` |
| Phone does not answer | 11 page cycles; access point first seen after 66 s, then repeated auth/disconnect |

### Test: 5 GHz through a changed NVS

Procedure:

1. Put a copy of the NVS with `dual_mode_select = 1` (byte 477) in tmpfs.
2. Point the `/var/run/wl1271-nvs.bin` symlink at the copy.
3. `rmmod wl12xx_sdio wl12xx`, then `insmod` both modules again.

Result: the unit did not come back on the network within 10 minutes and needed a reset. The
reset restored the stock symlink. The log was in tmpfs, so it was lost and the cause is not
known.

## Verified on

Firmware `cmu150_EU_74.00.324A`, legacy board.

## Open questions

- Does the board's RF front-end (FEM) have a 5 GHz path? The single-band profile may reflect
  the fitted FEM, not only a software setting.
- Why did the NVS test fail? A repeat needs a connection that does not go through `wlan0` and a
  log on persistent storage.
