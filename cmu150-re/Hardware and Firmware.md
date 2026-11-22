---
tags: [cmu150, hardware, firmware]
---

# Hardware and Firmware

Hardware platform, root filesystem and firmware update behaviour of the CMU150. Part of [[CMU150]].

## Unit
- **Mazda Connect CMU150** (Visteon / JCI generation), fitted to **MX-5 ND Gen1**.
- SoC: ARM (i.MX6-class). Kernel: **Linux 3.0.35**.
- Userland: **BusyBox v1.19.2** (built 2020-02-14), 274 applets — reference in [[BusyBox]].
- `JCI_OEM_PART_NUMBER = NA1P-669C0-E` (from `mfg-version.ini`, see
  [[config-mfg Keys Partition]]).

## I²C devices
| Bus-addr | `name` | Kernel driver | Function |
|---|---|---|---|
| `0-0018` | `aic310x` | `aic310x` | audio codec |
| `1-0020` | `adv7180` | `adv7180` | video decoder (camera input) |
| `2-0010` | `auth-ic` | none | Apple Authentication Co-Processor (MFi) |

- Bus `i2c-2` (`imx-i2c`); node `/dev/i2c-2`, `crw------- cmu root 89,2`.
- No kernel driver binds `auth-ic`; userspace talks to it through `/dev/i2c-2`
  (`I2C_SLAVE_FORCE`). The node is opened per transaction, not held open.
- Users: `/usr/bin/mfid` (SM service `mfid`, `autorun="yes"` in `jci/sm/sm.conf` and
  `jci/sm/sm_WCP.conf`; runs alongside `/usr/bin/carplayd` and SM service `jciCARPLAY`),
  `/usr/bin/ipoddev_full_auth`, `jci/lib/libjcii2ccp.so`.
- `/usr/bin/auth_ic_test`: stock diagnostic for the co-processor. Options: `-d <dev>`
  (default `/dev/i2c-2`), `-f` firmware version, `-p` protocol version, `-r` reset
  (writes the GPIO port directly via `/dev/mem`), `-q` quiet, `-w <ms>` delay between messages.
  Registers: firmware/protocol version, device ID, error code, control/status, signature,
  challenge, certificate data 1–15, self-test.
- Stock i2c-tools: `/usr/sbin/i2cdetect`, `i2cdump`, `i2cget`, `i2cset`.

## Board type and SM config selection
- `/usr/bin/autostart` (shell script) starts SM with one of two configs:
  `SMCFG_NORMALMODE=/jci/sm/sm.conf` or `SMCFG_WCPMODE=/jci/sm/sm_WCP.conf`, chosen by
  the exit code of `/jci/scripts/get_board_type.sh` (also linked as `/jci/bin/get_board_type.sh`).
- `get_board_type.sh` reads GPIOs `/sys/class/gpio/DI_TP1056/value` and `DI_TP1029/value`
  (3 retries, 1 s apart): either one `1` → exit `2` ("For WCP, New Hardware") → `sm_WCP.conf`;
  both `0` or GPIOs absent → exit `1` → `sm.conf`.
- Legacy board: `DI_TP1056=0`, `DI_TP1029=0`, SM runs `-f /jci/sm/sm.conf`.
- `sm_WCP.conf` vs `sm.conf` (13 differences):

| Service | `sm.conf` | `sm_WCP.conf` |
|---|---|---|
| `jciMMUI` args | `…display=filtered-touchscreen0` | `…, wcp` |
| `start_network` / `start_wifi` args | `UCP` | empty |
| `jciUSBMGR`, `jciCARPLAY`, `jciBCA`, `jcinavi`, `blm_nms_svc` args | none | `wcp` |
| `jciAAPA` args | none | `new_hw` |
| `bds` args | `-interface dbus` | `-interface dbus -wcp` |
| `jciLDS` args | `…lds.xml` | `…lds.xml GPS=UBLOX`, plus dependency `stage_navi` |
| one service | — | extra dependency `emmc_mount` |

  The WCP config also targets different hardware (u-blox GPS, `new_hw` for AA).

## Wi-Fi / Bluetooth
- FCC ID `CB262932` (Johnson Controls Interiors, label `MAZDA_GEN_65_CMU`, granted 2013-07-16): grants cover **2.4 GHz only**: Bluetooth 2402–2480 MHz, WLAN 2412–2462 MHz. No 5 GHz (UNII) grant. Internal photos (fccid.io exhibit 2017138) show **one** radio module (a TI WiLink combo chip on a castellated module under a shield can, the bare module and its can photographed removed) and **one** printed L-shaped trace antenna on the board edge. Wi-Fi and Bluetooth therefore share one chip and one antenna; the module carries a small single front-end IC, consistent with the single-band NVS profile below.
- TI WiLink `wl12xx` over SDIO (`SDIO_ID=0097:4076`, driver `wl1271_sdio`, backport
  `ol_R5.SP8.01`, firmware Rev 7.3.10.0.142); BT codec link `wl1285q-bt`.
- Client mode: `wpa_supplicant -c /data/nms/wpa_supplicant-wlan0.conf -i wlan0 -D nl80211`.
- Boot auto-connect: `blm_nms` (log tag `Generic`) enters `WiFiAutoConnecting` ~20 s after `NMS` starts; while no saved network is in the scan, `wpa_supplicant` rescans every ~5 s (`CTRL-EVENT-SCAN-STARTED` → `SCAN-RESULTS` → `NETWORK-NOT-FOUND`). The join time is then set by when a scan first contains the AP, not by the boot.
- Wi-Fi/BT coexistence (wl12xx combo chip, one antenna): while `BCA` pages the paired phone for HFP (`HFP connect start` → `Handsfree: Disconnected->Connecting`, 5 s `Page time out is occurred`, retry ~5 s later, in a loop while the phone does not answer), Wi-Fi scans return no BSS at all, even with the AP 2 m away. The first AP sighting comes in a gap between page attempts. Bench 2026-11-21: phone answered in 2 s → Wi-Fi connected 8 s after `WiFiAutoConnecting`; phone did not answer → 11 page cycles, AP first seen 66 s later, then repeated auth/disconnect.
- Weak-link symptoms in the log: scans that list no BSS at all, `SME: Trying to authenticate` without `Associated`, kernel `wl12xx: WARNING Unable to flush all TX buffers, timed out (timeout 500 ms`, and `CTRL-EVENT-DISCONNECTED ... reason=3 locally_generated=1` (the station leaves).
- Stock `/usr/sbin/iw` and `/usr/sbin/hostapd`. Interface modes include `AP`.
- `iw list`: Band 1 (2.4 GHz) has channels 1–14. Band 2 (5 GHz) is declared (HT20, MCS 0–7)
  but lists **no frequencies**. 5 GHz is unavailable with the stock driver config on the
  legacy board.
- Regulatory domain at runtime: `country 00` (world); its 5 GHz ranges are `PASSIVE-SCAN, NO-IBSS`.
- Cause of the empty 5 GHz band: the NVS calibration file has `dual_mode_select = 0`.
  The wl12xx driver enables 11a only when that flag is set; otherwise the 5 GHz band keeps zero channels.
  - `/lib/firmware/ti-connectivity/wl1271-nvs.bin` → `/var/run/wl1271-nvs.bin` →
    `/config-mfg/wl1271-nvs.bin` (1113 bytes, mode 0600, factory partition).
  - `/jci/scripts/check-config.sh` creates it only when it is absent:
    `calibrator plt autocalibrate wlan0 <wl12xx_sdio.ko> /usr/share/ti/wifi-utils/ini_files/128x/TQS_S_2.6.ini …`.
  - `TQS_S_2.6.ini` = TriQuint **single-band** front-end profile: `Single_Dual_Band_Solution = 00`,
    `FemVendorAndOptions = 01` (TQS), `XTALItrimVal = 04`, `Platform_configuration = 02`.
    Dual-band profiles ship alongside (`TQS_D_1.0.ini`, `TQS_D_1.7.ini`: `Single_Dual_Band_Solution = 01`) but are unused.
  - In the factory NVS, the general parameters start at byte offset 468:
    `xtal_itrim 0x04, platform_conf 0x02, dual_mode_select 0x00`.
- Whether the board's RF front-end (FEM) has a 5 GHz path is not established. The
  single-band profile choice may reflect the fitted FEM rather than a software limit.
- Runtime test (2026-11-22): an NVS copy in tmpfs with `dual_mode_select = 1` (byte 477), the `/var/run/wl1271-nvs.bin` symlink pointed at it, then `rmmod wl12xx_sdio wl12xx` + `insmod` of both. The unit did not come back on the network within 10 minutes and needed a reset; the reset restored the stock symlink. The cause is not known (the log was in tmpfs). A repeat needs a link that does not go through `wlan0` and a log on persistent storage.
- `wl12xx.ko` strings: NVS request `ti-connectivity/wl1271-nvs.bin`, and after each firmware boot the debug line `11a is %ssupported`. In the upstream wl12xx driver of this era the NVS is fetched once when the hardware registers, and the 5 GHz band's channel count is set to 0 at firmware boot when 11a is off. Both mean a changed NVS takes effect only after the module is reloaded (or the unit boots).

## Firmware
- `cmu150_EU_74.00.324A` (EU region). Compatible with the MX-5 ND Gen1: CX-5 and MX-5 share
  the CMU150 platform and the EU 74.xx line.
- Upgrade path: 74.00.311 → 74.00.324.
- On fw74 the `cmu` shell account's password login is disabled by default and the old `jci`
  default no longer matches; access is key-based. See [[config-mfg Keys Partition]] and
  [[SSH Access]].

## Root filesystem
- Root is **relfs, mounted read-only**. To write, remount it rw for the duration of the
  change, then back:
  - `mount -o rw,remount /` → edit → `mount -o ro,remount /` ([[mount]]).
  - The hardware watchdog can be paused while writing:
    `/sys/class/gpio/Watchdog Disable/value` = `1`, restore `0` afterwards.
- A **firmware update wipes the root fs**, so any root-fs edit must be re-applied after an
  OS update. `/data` is formatted too (see "Firmware update"); `/data_persist` survives.

## Firmware update
- `cmu150_EU_74.00.324A_update.up`: 2 432 208 800 bytes, MD5 `9b841bc807a5010a8652bb3b9e894f91`,
  SHA-1 `3aa55dadc8d59fadb0c69af17444447b097baf18`,
  SHA-256 `3a506745c4ae65c21e0427714f377a4765b92e339cdbd6bf1b91ec398bb1a963`.
- This package (`UPType = "Reinstall"`) carries and writes the fail-safe image (`fail-safe` step,
  version string `MAZ_CMU-150_74.00.324A`). The fail-safe version on a unit is a separate value
  and may be older if the unit was updated with a package without that step; read it with
  `/jci/scripts/jci-failsafe-version.sh` ([[Boot Chain and Fail-Safe]]).
- The `.up` update file is a zip (store method); some entries are encrypted.
- Root fs payload in `rootfs1upd/`: `execute.ini` copies `rootfs.tar.gz`,
  `rootfs.md5` and `update_rootfs.sh` to `/tmp` and runs the script.
- Entry names in `rootfs1upd/` (from `execute.ini`): `e0000000001.dat.gz` = `rootfs.tar.gz`,
  `e0000000002.dat.gz` = `rootfs.md5` (paths under `/tmp/rootfs/`),
  `e0000000003.dat.gz` = `update_rootfs.sh`. Each entry is gzip-wrapped, so
  `e0000000001.dat.gz` is gzip inside gzip: decompress twice to get the plain tar.
  The tar keeps numeric owners (`1000:1000`), modes and symlinks; extract it on a
  POSIX file system with `tar -xpf … --numeric-owner`.
- `update_rootfs.sh` (ver 2.0): if `/dev/ffx01p2` or `/dev/ffx01p3` exist, it
  repartitions `/dev/ffx01` into one partition (fdisk); then
  `mkfs.relfs /dev/ffx01p1 -NoConfirm`, mounts it rw at `/tmp/rootfs`,
  `tar xzvf rootfs.tar.gz`, `md5sum -c rootfs.md5`, `sync`, unmounts.
- The root fs is therefore rebuilt file by file (format + extract), not written
  as a block image. `rootfs.md5` lists 8464 files with their md5.
- `rootfs.tar.gz` (74.00.324A): 8464 regular files, 428,574,218 bytes, 920 dirs,
  737 symlinks; file mtimes span 1331059646 .. 1661846570 (682 distinct values).
  `tar x` keeps these mtimes, so after an update every stock file carries the
  archive's size and mtime.
- Step order (`main_instructions.ini`, `UPType = "Reinstall"`): cleandatapersist,
  bootstrap, ibc2, fail-safe, checksumoption, ibc1, linux1, getoldflavor,
  rootfs1upd, getnewflavor, passwdupdate, gnss, resources, usersettingsbackup,
  usersettingsrestore, usersettingscleanup, preloaddata, compactwnn,
  neutralizeid7, systemupdateid, vip.
- `/data` (`/dev/ffx01p4`), `retain_user_settings.sh` ver 1.1:
  - `--backup` (only when old and new flavor match, compared from char 8 on):
    copies `devmgr/ favorites/ nms/ save/ stationlogo/ itt/ nviback/ SXM/`,
    `com/jci/cpp/settings/*`, `lot_objects.xml`, `bt_addr_secondary.dat`,
    `smdb_vdm_*.dat`, `sms.cfg`, `TuneMixId*` to `/data_persist/reflash/backup/`.
  - `--restore`: always `mkfs.relfs /dev/ffx01p4`; then, if flavors match, copies
    the backup back; otherwise `/data` stays empty and the `nv-config` MTD is
    erased (`flash_erase`).
  - `--cleanup`: removes `/data_persist/reflash/backup/copy`.
  - Anything else under `/data` (e.g. `/data/integration/`) does not survive.
- `/data_persist` (`/dev/mmcblk0p2`) is not formatted: `clean_data_persist.sh`
  empties `log/dumps/` only when it is ≥150 MB; `preloaddata` copies
  `sirius-xm/` for the `cmu150_NA` flavor only; `systemupdateid` writes `1` to
  `SystemUpdateID.txt`.
- `/config-mfg` (squashfs, MTD `config`): `passwd_update.sh` copies the current
  contents, overwrites `passwd` and `authorized_keys` with the package's, rebuilds
  with `mksquashfs` and writes it back (`flash_erase` + `dd`).
