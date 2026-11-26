---
applet: "mdev"
tags: [busybox/command]
---

# `mdev`

```text
Usage: mdev [-s]

	-s	Scan /sys and populate /dev during system boot

It can be run by kernel as a hotplug helper. To activate it:
 echo /sbin/mdev > /proc/sys/kernel/hotplug
It uses /etc/mdev.conf with lines
[-]DEVNAME UID:GID PERM [>|=PATH] [@|$|*PROG]
```

---
[[BusyBox|← All BusyBox commands]]
