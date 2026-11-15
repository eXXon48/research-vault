---
tags: [cmu150, keys, security]
---

# config-mfg Keys Partition

The read-only per-unit keystore of the CMU150. Part of [[CMU150]]. See also
[[Partitions and Storage]].

- Device: `mtd5` ("config"), **squashfs** (gzip), mounted at `/config-mfg` read-only.
- The flash is not written during normal operation; the root fs redirects into it
  (`/etc/passwd` is a symlink to `/config-mfg/passwd`).

## Contents
- **`passwd`** — the real target of the `/etc/passwd` symlink:
  - `cmu` : salted **sha256crypt** hash : **uid 0 (root)** : home `/root` :
    shell `/bin/sh`.
  - `service` / `hmi` / `browser` : uid 1001–1003 : shell `/bin/false`.
- **`authorized_keys`** — a single `ssh-rsa` key (manufacturer dev key). Key
  material is not reproduced here.
- **`bt_addr.dat`** — Bluetooth MAC (per unit).
- **`wl1271-nvs.bin`** — TI WL1271 WiFi calibration / NVS (per unit).
- **`fgsn.dat`**, **`mfg-version.ini`** (`JCI_OEM_PART_NUMBER="NA1P-669C0-E"`).

## cmu password
- The stored hash is `$5$…` (sha256crypt, salted).
- The community default password `jci` does not match on fw74. On fw74 password login is
  disabled and access is through the `authorized_keys` dev key.
- Because `/etc/passwd` is a symlink, it can point at a writable copy on the root fs, so a
  password can be set without writing the MTD. See [[SSH Access]].
