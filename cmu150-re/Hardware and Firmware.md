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
- The `.up` update file is a zip (store method); some entries are encrypted.
- Root fs payload in `rootfs1upd/`: `execute.ini` copies `rootfs.tar.gz`,
  `rootfs.md5` and `update_rootfs.sh` to `/tmp` and runs the script.
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
