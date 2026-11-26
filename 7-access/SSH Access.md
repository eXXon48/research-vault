---
tags: [cmu150, ssh, security]
---

# SSH Access

What the stock firmware ships for SSH: the sshd configuration, the accounts it uses, how it starts at
boot, the firewall rules, and why root filesystem changes to it do not cause a boot loop.

## Overview

- sshd listens on port 36000, accepts public keys only, and reads authorized keys from the read-only
  `/config-mfg` partition.
- It starts once at boot (~25 s) with no retry. Port 36000 is inside the ephemeral port range, so
  sshd can fail to bind and stay down until the next boot.
- The firewall drops port 36000 on `wlan0`, so stock SSH works over USB Ethernet but not over Wi-Fi.
- The root filesystem has no integrity check. Changes to the SSH files do not trigger a reboot.

Related: [[config-mfg Keys Partition]], [[Users and Privileges]], [[Partitions and Storage]],
[[Hardware and Firmware]], [[README|CMU150 research]].

## Reference

### sshd configuration

`/etc/ssh/sshd_config` (OpenSSH):

| Setting | Value |
|---|---|
| `Port` | `36000` |
| `PermitRootLogin` | `yes` |
| `PubkeyAuthentication` | `yes` |
| `PasswordAuthentication` | `no` (blocks password login) |
| `AuthorizedKeysFile` | `/config-mfg/authorized_keys` |
| `StrictModes` | `no` |

- The firmware ships two byte-identical configs, `/etc/ssh/sshd_config` and `/etc/sshd_config`. sshd
  is started without `-f`, so which one is live depends on the build default; both are the same.
- There is no PAM (`/etc/pam.d` is absent): sshd checks the crypt hash in `/etc/passwd` directly.
  With `StrictModes no` and `PermitRootLogin yes`, only `PasswordAuthentication no` blocks a password
  login for `cmu`.

### Accounts

- `/etc/passwd` is a link to `/config-mfg/passwd` (read-only).
- There is no `/etc/shadow`: the password hash is in field 2 of `passwd`. The system `crypt()`
  supports sha256crypt.
- `cmu` is uid 0. The firmware 74 password is not the widely published default for older firmware.
- `/config-mfg/passwd` is `755 root:root`.

### Start at boot

- `/jci/scripts/start_network.sh` (SM service `start_network`, `retry_count="0"`) starts
  `/usr/sbin/sshd` once, unconditionally, about 25 s into boot.
- Port 36000 is inside the ephemeral port range (`/proc/sys/net/ipv4/ip_local_port_range` =
  `32768 61000`; `ip_local_reserved_ports` is empty). If a local TCP client holds 36000 as its source
  port at that moment, sshd fails and stays down until the next boot. The log shows:

  ```
  sshd[…] error: Bind to port 36000 on 0.0.0.0 failed: Address already in use.
  fatal: Cannot bind any address.
  ```

  The unit then answers "connection refused" on 36000. The error survives the boot in
  `/data_persist/log/error_logs/log`.
- Ports below 32768 (for example 22) are outside the ephemeral range and cannot be taken this way.
  Nothing in the stock rootfs listens on or references port 22 (only a commented `#   Port 22` in the
  client config `/etc/ssh/ssh_config`).

### Firewall

`/jci/scripts/jci-fw.sh` (stock, 5761 bytes) sets the iptables policy `INPUT DROP`, `FORWARD DROP`,
`OUTPUT ACCEPT`. Its SSH rules, in order:

1. `INPUT`: drop `tcp --dport 36000` arriving on `wlan0` (NEW, ESTABLISHED);
2. `INPUT`: accept `tcp --dport 36000` on any interface.

The first rule is why stock SSH works over USB Ethernet but not over Wi-Fi. `jci-fw.sh` has no rule
for port 22, so an sshd on 22 also needs an `ACCEPT` rule for `--dport 22`.

### Root filesystem changes and boot

- **No integrity check.** There is no dm-verity or signature enforcement, and the boot scripts contain
  none. Editing the passwd link, `sshd_config` and `jci-fw.sh` on the rootfs does not trip a check
  ([[Hardware and Firmware]]).
- **The `/etc/passwd` link is never regenerated** at boot. Nothing recreates it (only a comment in
  `config-update.sh` mentions it), so a changed link target persists.
- **`/jci/scripts/check-config.sh`** restores passwd only `if [ ! -f /config-mfg/passwd ]`: a presence
  check, with no hash or content validation, on the real `/config-mfg/passwd`, not on the
  `/etc/passwd` link. With the file present the block is skipped. It never reboots on a link change.
  The same presence-only logic guards `wl1271-nvs.bin`, `bt_addr.dat` and `fgsn.dat`.
- A malformed passwd copy breaks login; it does not cause a reboot.
- The rootfs is relfs, a persistent POSIX filesystem ([[Partitions and Storage]]). A file that keeps
  the stock mode and owner looks the same as stock across reboots.

## Measurements

| Item | Value |
|---|---|
| sshd bind failure on port 36000 | 2 of 4 boots in one bench session |

## Verified on

Firmware `cmu150_EU_74.00.324A`: stock rootfs for the files, bench unit for the boot behaviour.

## Open questions

- Which process holds port 36000 when sshd fails to bind.
