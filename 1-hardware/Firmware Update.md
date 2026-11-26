---
tags: [cmu150, firmware, update]
---

# Firmware Update

What the official `.up` update package contains, the steps it runs and what each partition
keeps afterwards.

## Overview

The official update is a USB `.up` file, a zip archive. It reinstalls the whole system: the
boot stages, the fail-safe image, the kernel, the root filesystem and the vehicle processor
firmware. The root filesystem is formatted and rebuilt from a tar archive file by file.
`/data` is formatted too; a fixed set of user settings is saved first and copied back.
`/data_persist` is not formatted. The user-side procedure is in the Revised docs guide
"Update to 74.00.324A".

Related: [[Hardware and Firmware]], [[Partitions and Storage]], [[Boot Chain and Fail-Safe]].

## Reference

### Package

| | |
|---|---|
| File | `cmu150_EU_74.00.324A_update.up` |
| Size | 2 432 208 800 bytes |
| MD5 | `9b841bc807a5010a8652bb3b9e894f91` |
| SHA-1 | `3aa55dadc8d59fadb0c69af17444447b097baf18` |
| SHA-256 | `3a506745c4ae65c21e0427714f377a4765b92e339cdbd6bf1b91ec398bb1a963` |
| Type | `UPType = "Reinstall"` |
| Format | zip (store method); some entries are encrypted |

### Update steps

Order from `main_instructions.ini`:

cleandatapersist, bootstrap, ibc2, fail-safe, checksumoption, ibc1, linux1, getoldflavor,
rootfs1upd, getnewflavor, passwdupdate, gnss, resources, usersettingsbackup,
usersettingsrestore, usersettingscleanup, preloaddata, compactwnn, neutralizeid7,
systemupdateid, vip.

The `fail-safe` step writes the fail-safe image (version string `MAZ_CMU-150_74.00.324A`). The
fail-safe version on a unit is a separate value. It can be older if the unit was updated with
a package that has no fail-safe step. Read it with `/jci/scripts/jci-failsafe-version.sh`
([[Boot Chain and Fail-Safe]]).

### Root filesystem payload (`rootfs1upd/`)

`execute.ini` copies three entries to `/tmp` and runs the script:

| Entry | Content | Path |
|---|---|---|
| `e0000000001.dat.gz` | `rootfs.tar.gz` | `/tmp/rootfs/` |
| `e0000000002.dat.gz` | `rootfs.md5` | `/tmp/rootfs/` |
| `e0000000003.dat.gz` | `update_rootfs.sh` | |

Each entry is gzip-wrapped, so `e0000000001.dat.gz` is gzip inside gzip. Decompress it twice
to get the plain tar.

`update_rootfs.sh` (version 2.0):

1. If `/dev/ffx01p2` or `/dev/ffx01p3` exist, repartition `/dev/ffx01` into one partition
   (fdisk).
2. `mkfs.relfs /dev/ffx01p1 -NoConfirm`.
3. Mount it read-write at `/tmp/rootfs`.
4. `tar xzvf rootfs.tar.gz`.
5. `md5sum -c rootfs.md5`.
6. `sync`, unmount.

So the root filesystem is formatted and extracted file by file, not written as a block image.

`rootfs.tar.gz` (74.00.324A):

| | |
|---|---|
| Regular files | 8464 (all listed with md5 in `rootfs.md5`) |
| Size | 428,574,218 bytes |
| Directories | 920 |
| Symlinks | 737 |
| File mtimes | 1331059646 .. 1661846570 (682 distinct values) |
| Owners | numeric `1000:1000` |

`tar x` keeps the mtimes. After an update every stock file therefore has the archive's size and
mtime.

### What each partition keeps

| Partition | What the update does |
|---|---|
| `/` (relfs, `/dev/ffx01p1`) | formatted and rebuilt from `rootfs.tar.gz` |
| `/data` (`/dev/ffx01p4`) | always formatted; a fixed set of user settings is copied back |
| `/data_persist` (`/dev/mmcblk0p2`) | not formatted |
| `/config-mfg` (squashfs, MTD `config`) | rebuilt with the package's `passwd` and `authorized_keys` |

#### `/data`: `retain_user_settings.sh` (version 1.1)

| Mode | Action |
|---|---|
| `--backup` | Runs only when the old and new flavor match (compared from character 8 on). Copies `devmgr/ favorites/ nms/ save/ stationlogo/ itt/ nviback/ SXM/`, `com/jci/cpp/settings/*`, `lot_objects.xml`, `bt_addr_secondary.dat`, `smdb_vdm_*.dat`, `sms.cfg`, `TuneMixId*` to `/data_persist/reflash/backup/`. |
| `--restore` | Always runs `mkfs.relfs /dev/ffx01p4`. If the flavors match, it copies the backup back. Otherwise `/data` stays empty and the `nv-config` MTD is erased (`flash_erase`). |
| `--cleanup` | Removes `/data_persist/reflash/backup/copy`. |

Anything else under `/data`, for example `/data/integration/`, is lost.

#### `/data_persist`

- `clean_data_persist.sh` empties `log/dumps/` only when it holds 150 MB or more.
- `preloaddata` copies `sirius-xm/`, for the `cmu150_NA` flavor only.
- `systemupdateid` writes `1` to `SystemUpdateID.txt`.

#### `/config-mfg`

`passwd_update.sh` copies the current contents and overwrites `passwd` and `authorized_keys`
with the package's. It then rebuilds the image with `mksquashfs` and writes it back
(`flash_erase` + `dd`).

### Versions shown in the HMI

**Settings → System → About → Version Information** opens a three-line dialog (`sysupdateApp`,
context `SoftwareInfo`):

| Line | On a 74.00.324A EU unit |
|---|---|
| `OS Version is <ver> <region/nav>` | `OS Version is 74.00.324 EU N` |
| `Music Database Version is <ver>` | `Music Database Version is 00.12.003` |
| `Fail Safe Version is <ver>`, or `Fail Safe Version Not Available` | `Fail Safe Version is 74.00.324` |

- The dialog leaves out the trailing `A` of the version.
- The strings come from `apps/sysupdate/js/sysupdateAppDict_en_US.js`.
- To open the dialog from the GUI: `Global.IntentSettingsTab {settingsTab:"System"}` →
  `syssettings SelectAbout` → `syssettings SelectVersion`.

## How to: extract the root filesystem from the package

1. Take `rootfs1upd/e0000000001.dat.gz` from the `.up` zip.
2. Decompress it twice (gzip inside gzip).
3. Extract the tar on a POSIX filesystem to keep owners, modes and symlinks:
   `tar -xpf … --numeric-owner`.

## Verified on

`cmu150_EU_74.00.324A_update.up`, legacy board.
