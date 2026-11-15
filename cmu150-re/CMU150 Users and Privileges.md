---
tags: [cmu150, security, users]
---

# CMU150 Users and Privileges

Who owns the stock files and which uids the running processes use. Firmware `74.00.324A`.
Related: [[SSH Access]], [[config-mfg Keys Partition]].

## Accounts
- `/etc/passwd` → `/config-mfg/passwd` (squashfs, RO): `cmu` uid 0, `service` 1001,
  `hmi` 1002, `browser` 1003 (the last three have shell `/bin/false`).
- `/etc/group` has `user:x:1000:`. **No passwd entry for uid 1000.**

## File ownership (rootfs image)
- 10121 of 10122 rootfs entries are owned **`1000:1000`**: `/jci`, `/etc`, `/bin`, `/sbin`,
  `/usr`, `/lib` and the top-level dirs. The one exception is `lost+found` (`0:0`).
- **No setuid files** on the rootfs.
- uid 1000 is the owner of the whole stock firmware, not an unprivileged user.
  It has no root capabilities. On any read-write mount, however, a uid-1000 process can
  overwrite stock files that root processes execute. This includes the rootfs once it is
  remounted rw.

## Running processes
- 191 processes run as **uid 0**. One runs as 1001, one as 1002. **No process runs as 1000.**
- The HMI browser (`opera`) runs as **uid 0**.
- `/dev/spidev1.0` and `/dev/spidev3.0` are `0:0 crw-------`, accessible to root only.

## Device nodes and D-Bus
- `/dev` is a **devtmpfs, mounted rw** (RAM). Node owner and mode can be changed at
  runtime, and the change is lost on reboot.
- `/dev/uinput` and `/dev/input/event0`–`event5` are `0:0 crw-------` (root only).
- `/dev/snd/*` (`controlC*`, `pcmC*`) are `0:63 crw-rw----`. Group 63 is `audio` in
  `/etc/group`, so raw ALSA access works for any uid that has gid 63.
- System bus (`/etc/dbus-1/system.conf`): the default policy denies `send_type="method_call"`
  and denies owning names. `system.d/` holds one policy, `<policy user="root">`. Method
  calls to system-bus services (Audio Manager, SafeReboot, etc.) are therefore available
  to root only. The policy files are on the read-only rootfs.
