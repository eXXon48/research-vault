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

## Normal-mode kernel and initramfs (verified, read-only dump of `/dev/ffx00`)
- `/dev/ffx00` (FlashFX disk 0, 22 908 928 bytes, no partition table) starts with the magic
  `TOOB`; a zImage starts at offset 512 (2 404 776 bytes) and a gzip initramfs at offset
  2 405 888 (newc cpio, 3 684 864 bytes unpacked, 55 entries). The kernel unpacks it at ~0.92 s
  ("Freeing initrd memory: 1540K"). The `.up` component that writes it is `linux1`.
- The initramfs `/init` is an ELF binary (7 564 bytes). It holds the Datalight modules
  `ffxos`, `flashfx`, `ffxblk`, `relos`, `reliance`, `relfs` under
  `lib/modules/3.0.35/kernel/datalight/`, plus `com_jci.ko`, `com_jci_cpp_nvram.ko` and the LVDS
  driver. These load at ~1.48 s, before `ffx01p1` is mounted as root (~2.5 s).
- The FlashFX modules there are byte-identical (md5) to the copies in the rootfs under
  `/lib/modules/3.0.35/kernel/drivers/datalight/`; the boot uses the initramfs copies.
- Module ABI: `vermagic=3.0.35 SMP preempt mod_unload ARMv7`, no `__versions` (no
  MODVERSIONS CRCs). `flashfx.ko` depends only on `ffxos`; every hardware access of
  `flashfx.ko` goes through the 75 symbols `ffxos.ko` exports (`FfxHookNTMX6*`, `FfxDclOs*`).
- Header (bytes 64–127, u32 LE): three entries `{type, 20, size, sector}` — 1 = zImage
  (2 404 776 B at sector 1), 2 = kernel command line (125 B at sector 4698), 1 = initramfs
  (1 579 252 B gzip at sector 4699). Command line: `console=ttymxc1,115200 lpj=8495104
  ldb=sin0 loglevel=3 usbcore.authorized_default=0 panic=23 mxc_vpu_mempool.max_instances=3`.
- The rest of the disk (from ~3.99 MB to 22.9 MB, ~18.9 MB) is all `0xFE`: unused.
- Kernel: `panic=23` (reboot 23 s after a panic, from the command line), `panic_on_oops=0`;
  `kallsyms_lookup_name` exported; no kprobes.
- Out-of-tree modules with a matching vermagic build against NXP `linux-imx` tag
  `rel_imx_3.0.35_4.1.0_ga_caf` with gcc 4.9.4 (`arm-linux-gnueabi`) and the stock config
  from `configs.ko`. The stock board option `CONFIG_MACH_MX6Q_CMU` does not exist in the NXP tree;
  without a replacement `oldconfig` drops `ARCH_MX6Q` and with it `SMP`. Substituting
  `CONFIG_MACH_MX6Q_SABREAUTO=y` keeps `SMP`; the result differs only in board-specific drivers.
  `.gnu.linkonce.this_module` is 0x14c bytes with `init`/`exit` at 0xbc/0x144, identical to
  stock modules, so `struct module` layout matches. A module built this way loads and unloads
  cleanly on the unit (insmod/rmmod, verified).

Not verified here (community claims): IBC2 load address OCRAM `0x00912000`, entry `0x009126E0`;
IBC2 building a device tree from the header; HABv4 not enabled.
