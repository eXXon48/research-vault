---
tags: [cmu150, boot, nor, failsafe, reflash]
---

# Boot Chain and Fail-Safe

SPI-NOR layout, the `boot-select` flag, the fail-safe image format and who writes the flag.
Firmware `cmu150_EU_74.00.324A`. Part of [[CMU150]]; storage overview in [[Partitions and Storage]],
SM-stage timing in [[Boot Timeline]].

Sources: kernel log (`m25p80` partition table, kernel command line), the official `.up`
(`fail-safe/`, `ibc1/`, `ibc2/`, `bootstrap/` entries), the stock root filesystem and the
fail-safe initramfs. Community documentation (silverchris `mazda-cmu-documentation`) agrees
with every value below that is marked verified.

## SPI-NOR layout (verified)
Chip: `m25p80 spi0.0: found mx25l6405d, expected s25fl064p` → Macronix MX25L6405D, 8 MiB,
64 KiB erase sectors. 9 MTD partitions, from the kernel log:

| MTD | Name | Flash range | Size | Content |
|---|---|---|---|---|
| `mtd0` | `bootstrap` | `0x000000–0x010000` | 64 KiB | first-stage loader |
| `mtd1` | `boot-select` | `0x010000–0x020000` | 64 KiB | boot selector flag (byte 0) |
| `mtd2` | `ibc1` | `0x020000–0x040000` | 128 KiB | IBC1 (normal-mode boot code) |
| `mtd3` | `ibc2` | `0x040000–0x060000` | 128 KiB | IBC2 (fail-safe boot code) |
| `mtd4` | `nv-config` | `0x060000–0x070000` | 64 KiB | erased by the reflash when the flavor changes |
| `mtd5` | `config` | `0x070000–0x0D0000` | 384 KiB | squashfs → `/config-mfg` |
| `mtd6` | `jci-boot-diag` | `0x0D0000–0x0E0000` | 64 KiB | — |
| `mtd7` | `fail-safe` | `0x0E0000–0x7E0000` | 7 MiB | fail-safe kernel + initramfs |
| `mtd8` | `update` | `0x7E0000–0x800000` | 128 KiB | reflash state storage |

- `boot-select` byte 0 is SPI-NOR absolute `0x010000`; `/dev/mtd1` offset 0 is the same byte.
- `.up` `binary.ini` files carry a `DeviceName` that does not always match this kernel
  (`fail-safe` says `/dev/mtd6`, `ibc1` `/dev/mtd2`, `ibc2` `/dev/mtd3`). `PartitionName` is
  the stable key; every stock script resolves MTDs by name.
- Normal-mode kernel command line includes `ibc=1.68.21 ibs=1.68.10 BOOTSEL=1
  ibcinfo=0x00929000` (boot code passes the selected IBC as `BOOTSEL`).

## `boot-select` flag (verified)
- Values: erased `0xFF` = normal boot (IBC1), `0x00` in byte 0 = fail-safe boot (IBC2).
  `libjcireflashua.so` reads the flag ("Boot select flag = %02X") and rejects other values
  ("The value of the boot select flag = %02X is wrong!").
- Stock writer: `/usr/sbin/switch_ibc` (shell, 3979 bytes, identical in the root fs and in the
  fail-safe initramfs). Same logic as `switch_ibc()` in `/usr/bin/configure_flash.sh`:
  - resolves the MTD by name `boot-select` via `/sys/class/mtd/mtd*/name`;
  - refuses unless `mtdinfo` reports type `nor`;
  - `switch_ibc 1` → `flash_erase <mtd> 0 0` (whole partition back to `0xFF`);
  - any other argument → `dd if=/dev/zero of=<mtd> bs=1 count=1` (byte 0 = `0x00`);
  - forces a dry run (`DEBUG=1`) when `uname -m` is not `armv7l`.
- Callers:

| Caller | Call | When |
|---|---|---|
| `svcjciupdatea.so` (normal-mode updater, root fs) | `switch_ibc 2` | an install step must continue in the fail-safe OS ("Discontinue the installation … could continue in Fail-Safe OS") |
| `reflash_app` (fail-safe initramfs) | `switch_ibc 1` | `reflash_app_SwitchToNormalMode`, at the end of a fail-safe reflash |
| `configure_flash.sh` (root fs, manual tool) | `switch_ibc 1` | end of its flash sequence |

- Guards in the stock callers: the normal-mode updater checks that the fail-safe image is
  present before switching ("The Fail-Safe OS image is not fully present to reboot in it!",
  `IsPresentFailSafeOS`); `reflash_app` checks the normal image before switching back
  ("The Normal OS image is not fully present to reboot in it!").
- Reflash state lives outside the flag: `[StateStorage]` in `um_app.ini` (normal) and
  `reflash.ini` (fail-safe) = NOR partition `update`, offset `0x0`, size `0x20000`.
  `[BootSelect] PartitionName = "boot-select"`, `FlashType = NOR`.
- No `.up` reflash step writes `boot-select` directly: `update-bootstrap.sh` writes only
  `bootstrap`; `ibc1`/`ibc2`/`fail-safe` are image copies to their own partitions.

## Fail-safe image format (verified on the 74.00.324A image)
`fail-safe/00000000.dat.gz` in the `.up` = single gzip → 7 340 032 bytes = the whole `fail-safe`
partition. 512-byte header, little-endian `uint32`:

| Offset | Value | Meaning |
|---|---|---|
| `0x00` | `54 4F 4F 42` ("TOOB", `0x424F4F54` read LE = "BOOT") | magic |
| `0x04` | `3` | record count |
| `0x08–0x43` | 0 | padding |
| `0x44` + 20·n | `{0x14, size, offset_blocks, blocks, id}` | records |

Records in 74.00.324A (offsets in 512-byte blocks from image start):

| id | Content | Size (bytes) | Offset (blk) | Blocks |
|---|---|---|---|---|
| `2` | kernel | 2 205 784 | 1 | 4309 |
| `1` | kernel command line | 119 | 4310 | 1 |
| `0` | initramfs (gzip cpio, newc, 225 members) | 5 022 172 | 4311 (`0x21AE00`) | 9809 |

- Fail-safe command line: `console=ttymxc1,115200 rdinit=/sbin/init ldb=sin0 quiet mem=950MB
  JCI_SW_VER=MAZ_CMU-150_74.00.324 JCI_SW_VER_PATCH=A`.
- The last 256 bytes of the partition start with a version string; offset 7 339 776 holds
  `MAZ_CMU-150_74.00.324A` (22 bytes). `/jci/scripts/jci-failsafe-version.sh` reads exactly
  these bytes (fast path), else the command line via `ibcmkimage -x -c`, else `jci/version.ini`
  from the initramfs. A cheap presence/version check of `mtd7` is therefore: magic at offset 0
  and `MAZ` at offset 7 339 776.
- A bench unit updated with the 74.00.324A EU `Reinstall` package reports
  `failsafe version: MAZ_CMU-150_74.00.324A` from `jci-failsafe-version.sh`, so that package
  does write `mtd7`.
- Image tool: `ibcmkimage` (`-x -c` extracts the command line, `-x -i` the initramfs).

## Fail-safe userspace (verified, initramfs `/sbin/init`)
1. Mounts `/dev`, `/proc`, `/sys`, tmpfs; loads `ffxblk`, `relfs`, MMC, `cmu_io`, GPIO, `uinput`.
2. Starts `lvdsLite` (display) and `udevd`.
3. Mounts eMMC: `mmcblk0p1` → `/mnt/resources`, `mmcblk0p2` → `/mnt/data_persist`
   (`/sbin/emmc_mount.sh`). Reflash log and backup folder are under `/mnt/data_persist/reflash`.
4. Starts `vim_app` (`-C1` or `-C0` from `/mnt/data_persist/checksum_result.txt`).
5. Runs `/jci/reflash/reflash_app -c /jci/reflash/reflash.ini` (blocks until done).
6. Re-enables the watchdog (`Watchdog Disable` = 0), unmounts relfs, unloads Datalight
   modules, resets the board via `/sys/class/gpio/Reset/value` = 1. With `/tmp/debug_flag`
   present it does not reset.
- `reflash_app` strings also mention: `jci-looptest` flag file (auto-reboot), "ACC cycle has been
  detected - Starting to prepare board for safe reboot", "A custom UP was found and executed on
  application start-up => safely reboot to Normal mode".
- USB package discovery in fail-safe uses the same `jci-autoupdate` + single `*.up` convention as
  normal mode (same strings and regex `.+[.]up$` in `reflash_app`).

## Boot chain (partly verified)
Bootstrap (`mtd0`) → reads `boot-select` → IBC1 (`mtd2`, normal) or IBC2 (`mtd3`, fail-safe) →
Linux 3.0.35. Normal mode: kernel → `init_cmu` → `/usr/bin/autostart` →
`start_normal_mode.sh` → SM (`/jci/sm/sm`). Fail-safe: kernel + initramfs from `mtd7` →
`/sbin/init` above.

Not verified here (community claims): IBC2 load address OCRAM `0x00912000`, entry `0x009126E0`;
IBC2 building a device tree from the header; HABv4 not enabled.
