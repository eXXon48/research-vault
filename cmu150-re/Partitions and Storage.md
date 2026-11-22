---
tags: [cmu150, storage, mtd]
---

# Partitions and Storage

MTD layout, mounts and writable surfaces of the CMU150. Part of [[CMU150]].

Sources for the layout: `/proc/partitions`, `/proc/mtd`, `df -h`, `/proc/mounts` and
`blockdev --getsize64` per device ([[blockdev]], [[df]]).

## Layout
- **MTD NOR/NAND flash** holds the system partitions (`/proc/mtd`). SPI-NOR (8 MiB, `mtd0`–`mtd8`:
  bootstrap, boot-select, ibc1, ibc2, nv-config, config, jci-boot-diag, fail-safe, update) with
  offsets and the `boot-select` flag: [[Boot Chain and Fail-Safe]].
- **`/config-mfg`** = `mtd5` ("config"), a **squashfs** (gzip). The partition is
  ~384 KB but only ~1167 bytes are used. **Read-only**; per-unit keys and
  calibration. Never written — see [[config-mfg Keys Partition]].
- **Root fs** `/` = `/dev/ffx01p1`, type **relfs** (Datalight Reliance Nitro on
  NAND via FlashFX), mounted `ro,noatime`; df ~602 MB. It is a real
  **transactional, persistent, POSIX read-write** filesystem — `mount -o
  rw,remount /` makes it writable and every change (**content, mode via chmod,
  owner via chown**) persists across reboot. Not an overlay/tmpfs. See
  [[Hardware and Firmware]].
- **`/data`** = persistent, writable.

## Writable surfaces, by persistence
| Mount | Writable | Survives reboot | Survives fw update |
|-------|----------|-----------------|--------------------|
| `/` (relfs) | only after `mount -o rw,remount /` | yes | **no** |
| `/config-mfg` (mtd5) | no (read-only squashfs) | n/a | yes, except `passwd` and `authorized_keys` (replaced) |
| `/data` (`/dev/ffx01p4`) | yes | yes | **no**: formatted; only a fixed set of user-settings items is copied back |
| `/data_persist` (`/dev/mmcblk0p2`) | yes | yes | yes (only `log/dumps/*` cleared when ≥150 MB) |
| `/tmp` (tmpfs, `size=95232k` ≈ 93 MB) | yes | no (RAM) | no |

Details of what the `.up` reinstall does to each partition: [[Hardware and Firmware]]
(section "Firmware update").

## Stock writers on the persistent partitions
Both writable flash partitions are written routinely by stock firmware:

- **`/data_persist/log/`** (eMMC): `/jci/scripts/log-save.sh` (run by the SM
  `report` hook `log-and-data-save` in `/jci/sm/sm.conf`) archives
  `/var/log/running_log` into `saved_logs/logsNNN_coreNNN_<ts>-<ver>.tar.bz2`
  (counter in `saved_logs/count.txt`, total capped at 50 MB). Core dumps land in
  `dumps/*.core.bz2` (`dumps/count.bin`); also `error_logs/`, `saved_resource_data/`.
  Typical size on a used unit: ~32 MB in `log/` (dumps ~22 MB, saved_logs ~7 MB).
  `data-save.sh` runs right after `log-save.sh`.
- **`/data/save/`** (NAND): navigation state as A/B double-buffered files with
  `.cks` checksum siblings (`dr_saved_values_A.bin`, `trafficevents_A.txt`,
  `profiles/01/driving_log_*`, `preloads/*.prl`), rewritten during normal runtime.

`/tmp` is a RAM-backed tmpfs (`none /tmp tmpfs rw,relatime,size=95232k`); data there
counts against system RAM (~764 MB total, ~230 MB free under the HMI). `/var` is a
symlink to `/tmp/var`. `/data` and `/data_persist` are mounted under `/tmp/mnt/`
(`/dev/ffx01p4` ≈ 307 MB, `/dev/mmcblk0p2` ≈ 572 MB) and reached via `/data`,
`/data_persist`.
