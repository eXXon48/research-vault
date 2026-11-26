---
tags: [cmu150, storage, mtd]
---

# Partitions and Storage

Flash layout, mounts and writable locations of the CMU150, and what survives a reboot or a
firmware update.

## Overview

System partitions live on an 8 MiB SPI-NOR flash (MTD devices). The root filesystem is on
NAND, behind the FlashFX driver, and uses the transactional relfs filesystem, mounted
read-only. Two writable flash partitions persist across reboots: `/data` on NAND and
`/data_persist` on eMMC. Only `/data_persist` survives a firmware update unchanged. `/tmp`
is RAM.

Sources: `/proc/partitions`, `/proc/mtd`, `df -h`, `/proc/mounts` and `blockdev --getsize64`
for each device ([[blockdev]], [[df]]).

## Reference

### Devices and mounts

| Mount | Device | Filesystem | Size | Mounted |
|---|---|---|---|---|
| `/` | `/dev/ffx01p1` | relfs (Datalight Reliance Nitro on NAND through FlashFX) | ~602 MB (df) | `ro,noatime` |
| `/config-mfg` | `mtd5` ("config") | squashfs (gzip) | ~384 KB partition, ~1167 bytes used | read-only |
| `/data` | `/dev/ffx01p4` | relfs | ≈ 307 MB | read-write |
| `/data_persist` | `/dev/mmcblk0p2` | (eMMC) | ≈ 572 MB | read-write |
| `/tmp` | tmpfs | tmpfs | `size=95232k` ≈ 93 MB | read-write |

- SPI-NOR (8 MiB, `mtd0`–`mtd8`): bootstrap, boot-select, ibc1, ibc2, nv-config, config,
  jci-boot-diag, fail-safe, update. Offsets and the `boot-select` flag:
  [[Boot Chain and Fail-Safe]].
- `/data` and `/data_persist` are mounted under `/tmp/mnt/` and reached through the `/data`
  and `/data_persist` paths.
- `/var` is a symlink to `/tmp/var`.
- `/tmp` is RAM: data there counts against system RAM (~764 MB total, ~230 MB free with the
  HMI running). The mount line is `none /tmp tmpfs rw,relatime,size=95232k`.

### Root filesystem

relfs is a transactional, persistent, POSIX read-write filesystem. It is not an overlay and
not tmpfs. After `mount -o rw,remount /` the root is writable, and every change persists
across reboot: content, mode (chmod) and owner (chown). Procedure:
[[Hardware and Firmware]] ("How to: write to the root filesystem").

### `/config-mfg`

Per-unit keys and calibration, read-only, never written during normal operation:
[[config-mfg Keys Partition]].

### Persistence

| Mount | Writable | Survives reboot | Survives firmware update |
|---|---|---|---|
| `/` (relfs) | only after `mount -o rw,remount /` | yes | **no** |
| `/config-mfg` (mtd5) | no (read-only squashfs) | n/a | yes, except `passwd` and `authorized_keys` (replaced) |
| `/data` (`/dev/ffx01p4`) | yes | yes | **no**: formatted; only a fixed set of user settings is copied back |
| `/data_persist` (`/dev/mmcblk0p2`) | yes | yes | yes (only `log/dumps/*` cleared when ≥150 MB) |
| `/tmp` (tmpfs) | yes | no (RAM) | no |

Details of what the update does to each partition: [[Firmware Update]].

### Stock writers on the persistent partitions

Stock firmware writes both writable flash partitions during normal operation.

**`/data_persist/log/`** (eMMC):

- `/jci/scripts/log-save.sh` runs from the SM `report` hook `log-and-data-save` in
  `/jci/sm/sm.conf`. It archives `/var/log/running_log` into
  `saved_logs/logsNNN_coreNNN_<ts>-<ver>.tar.bz2`. The counter is in `saved_logs/count.txt`;
  the total is capped at 50 MB.
- `data-save.sh` runs right after `log-save.sh`.
- Core dumps go to `dumps/*.core.bz2` (counter `dumps/count.bin`).
- Other directories: `error_logs/`, `saved_resource_data/`.
- Typical size on a used unit: ~32 MB in `log/` (dumps ~22 MB, saved_logs ~7 MB).

**`/data/save/`** (NAND): navigation state, rewritten during normal runtime. The files are
A/B double-buffered, each with a `.cks` checksum sibling: `dr_saved_values_A.bin`,
`trafficevents_A.txt`, `profiles/01/driving_log_*`, `preloads/*.prl`.

## Verified on

Firmware `cmu150_EU_74.00.324A`, legacy board.
