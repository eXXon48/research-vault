---
tags: [cmu150, security, users]
---

# Users and Privileges

Which accounts exist, who owns the stock files, which uids the running processes use, and who can
open devices and call D-Bus services.

## Overview

Almost everything runs as root. The whole stock firmware is owned by uid 1000, which has no
`/etc/passwd` entry and no running process. Devices and system-bus method calls are root only,
except ALSA, which is open to group `audio`.

Related: [[SSH Access]], [[config-mfg Keys Partition]].

## Reference

### Accounts

`/etc/passwd` is a link to `/config-mfg/passwd` (squashfs, read-only):

| Account | uid | Shell |
|---|---|---|
| `cmu` | 0 | — |
| `service` | 1001 | `/bin/false` |
| `hmi` | 1002 | `/bin/false` |
| `browser` | 1003 | `/bin/false` |

`/etc/group` has `user:x:1000:`. There is no passwd entry for uid 1000.

### File ownership (rootfs image)

- 10121 of 10122 rootfs entries are owned by `1000:1000`: `/jci`, `/etc`, `/bin`, `/sbin`, `/usr`,
  `/lib` and the top-level directories. The one exception is `lost+found` (`0:0`).
- There are no setuid files on the rootfs.
- uid 1000 owns the whole stock firmware; it is not an unprivileged user. It has no root
  capabilities. On any read-write mount, however, a uid-1000 process can overwrite stock files that
  root processes execute. This includes the rootfs once it is remounted read-write.

### Running processes

| uid | Processes |
|---|---|
| 0 | 191, including the HMI browser `opera` |
| 1001 | 1 |
| 1002 | 1 |
| 1000 | none |

### Device nodes

| Node | Owner, mode | Access |
|---|---|---|
| `/dev/uinput`, `/dev/input/event0`–`event5` | `0:0 crw-------` | root only |
| `/dev/spidev1.0`, `/dev/spidev3.0` | `0:0 crw-------` | root only |
| `/dev/snd/*` (`controlC*`, `pcmC*`) | `0:63 crw-rw----` | root and gid 63 (`audio` in `/etc/group`) |

`/dev` is a devtmpfs, mounted read-write (RAM). Node owner and mode can be changed at runtime; the
change is lost on reboot.

### D-Bus

**Standard system bus** (`/etc/dbus-1/system.conf`):

- The default policy denies `send_type="method_call"` and denies owning names.
- `system.d/` holds one policy, `<policy user="root">`.

Method calls on the standard system bus are therefore available to root only.

**JCI service and HMI buses** (`/tmp/dbus_service_socket`, `/tmp/dbus_hmi_socket`): separate
`dbus-daemon` instances with their own configuration (`/etc/dbus-1/service.conf`,
`service.d/jci_service.conf`, `hmi.d/jci_hmi.conf`). They have `<allow_anonymous/>` and an
allow-all default policy ([[Audio Source Registration]], "D-Bus surface";
[[SM Service Graph]], "D-Bus").

All policy files are on the read-only rootfs.

## Verified on

Firmware `cmu150_EU_74.00.324A`: stock rootfs image for file ownership, bench unit for processes and
device nodes.

## Open questions

- The socket owner and mode of the JCI service and HMI buses, and whether a non-root process can
  connect to them.
