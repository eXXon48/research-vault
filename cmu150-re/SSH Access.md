---
tags: [cmu150, ssh, security]
---

# SSH Access

What the stock firmware ships for SSH, and why root-fs changes to it do not cause a boot loop.
Part of [[CMU150]]. See also [[config-mfg Keys Partition]].

## What the firmware ships
`/etc/ssh/sshd_config` (OpenSSH):
- `Port 36000`
- `PermitRootLogin yes`
- `PubkeyAuthentication yes`
- `PasswordAuthentication no`  ← blocks password login by default
- `AuthorizedKeysFile /config-mfg/authorized_keys`

`sshd` is started once, with no retry, by `/jci/scripts/start_network.sh` (SM service
`start_network`, `retry_count="0"`), about 25 s into boot. Port 36000 lies **inside the
ephemeral port range** (`/proc/sys/net/ipv4/ip_local_port_range` = `32768 61000`;
`ip_local_reserved_ports` is empty). If a local TCP client holds 36000 as its source port at that
moment, sshd fails and does not come back until the next boot. In the log the failure looks like this:
`sshd[…] error: Bind to port 36000 on 0.0.0.0 failed: Address already in use.` →
`fatal: Cannot bind any address.` (seen on 2 of 4 boots on 2026-11-20, at 16:19 and 18:10; the unit
then answers "connection refused" on 36000). Which process held the port is not established.
Ports below 32768 (e.g. 22) are outside the ephemeral range and cannot be taken this way. Nothing in
the stock rootfs listens on or references port 22 (only a commented `#   Port 22` in
`/etc/ssh/ssh_config`, the client config). `jci-fw.sh` has no rule for 22, so with policy
`INPUT DROP` a sshd on 22 also needs an `ACCEPT` rule for `--dport 22`. The error survives the boot in `/data_persist/log/error_logs/log`.

Account facts:
- `/etc/passwd` is a **symlink → `/config-mfg/passwd`** (read-only).
- There is **no `/etc/shadow`** — the password hash lives in field 2 of
  `passwd`. The system `crypt()` supports sha256crypt.
- `cmu` is **uid 0** (already root). The fw74 password is not the `jci` community default.
- The firmware ships **two byte-identical** configs: `/etc/ssh/sshd_config` and
  `/etc/sshd_config`. sshd is started with **no `-f`** (`/usr/sbin/sshd` from
  `/jci/scripts/start_network.sh`, unconditional at boot), so the live one is
  ambiguous — both exist.
- **No PAM** (`/etc/pam.d` absent) → sshd matches the crypt hash directly from
  `/etc/passwd`. `StrictModes no` + `PermitRootLogin yes` → nothing else blocks
  the cmu password login. Only `PasswordAuthentication no` stands in the way.

## Boot-time behaviour
- **No rootfs integrity check.** The root fs can be modified and still boots: there is no
  dm-verity / signature enforcement, and the boot scripts contain none. Editing the passwd
  symlink, `sshd_config` and `jci-fw.sh` on the root fs does not trip a check. See
  [[Hardware and Firmware]].
- **The `/etc/passwd` symlink is never regenerated** at boot — nothing recreates
  it (only a comment mentions it in `config-update.sh`), so a repoint persists.
- **`/jci/scripts/check-config.sh`** restores passwd only `if [ ! -f
  /config-mfg/passwd ]` — a *presence* check, no hash/content validation, acting
  on the real `/config-mfg/passwd`, not the `/etc/passwd` symlink. With the file
  present the block is skipped; it never reboots on a symlink change. (Same
  presence-only logic guards wl1271-nvs.bin, bt_addr.dat, fgsn.dat.)
- A malformed passwd copy breaks login; it does not cause a reboot.
- **Perms/owner.** `/config-mfg/passwd` is **755 root:root**. Root is relfs
  (persistent POSIX fs, see [[Partitions and Storage]]), so a file that preserves
  mode/owner is indistinguishable from stock across reboot.

## Firewall
`/jci/scripts/jci-fw.sh` (stock, 5761 bytes) sets the iptables policy `INPUT DROP`,
`FORWARD DROP`, `OUTPUT ACCEPT`. Its SSH rules, in order: an `INPUT` rule dropping
`tcp --dport 36000` arriving on `wlan0` (NEW, ESTABLISHED), then an `INPUT` rule accepting
`tcp --dport 36000` on any interface. The first rule is why stock SSH works over USB
ethernet but not over WiFi.
