---
tags: [cmu150, keys, security]
---

# config-mfg Keys Partition

The read-only per-unit keystore of the CMU150: accounts, the SSH key and radio calibration.

## Overview

`/config-mfg` is a small squashfs image on the `mtd5` ("config") flash partition, mounted
read-only. Normal operation never writes it. The root filesystem points into it with
symlinks: `/etc/passwd`, for example, is a symlink to `/config-mfg/passwd`. A firmware update
rebuilds the image and replaces `passwd` and `authorized_keys` ([[Firmware Update]]).

Related: [[Partitions and Storage]], [[SSH Access]].

## Reference

| | |
|---|---|
| Device | `mtd5` ("config") |
| Filesystem | squashfs (gzip) |
| Mount | `/config-mfg`, read-only |

### Contents

| File | Content |
|---|---|
| `passwd` | target of the `/etc/passwd` symlink (accounts below) |
| `authorized_keys` | one `ssh-rsa` key, the manufacturer development key (key material not reproduced here) |
| `bt_addr.dat` | Bluetooth MAC address (per unit) |
| `wl1271-nvs.bin` | TI WL1271 Wi-Fi calibration (NVS), per unit ([[Wi-Fi and Bluetooth]]) |
| `fgsn.dat` | per-unit data |
| `mfg-version.ini` | `JCI_OEM_PART_NUMBER="NA1P-669C0-E"` |

### Accounts in `passwd`

| Account | Password | uid | Home | Shell |
|---|---|---|---|---|
| `cmu` | salted sha256crypt hash (`$5$…`) | **0 (root)** | `/root` | `/bin/sh` |
| `service`, `hmi`, `browser` | | 1001–1003 | | `/bin/false` |

### `cmu` password

- On firmware 74 the community default password `jci` does not match.
- Password login is disabled; access goes through the development key in `authorized_keys`.
- `/etc/passwd` is a symlink, so it can point at a writable copy on the root filesystem. That
  lets you set a password without writing the MTD ([[SSH Access]]).

## Verified on

Firmware `cmu150_EU_74.00.324A`.
