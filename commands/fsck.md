---
applet: "fsck"
tags: [busybox/command]
---

# `fsck`

```text
Usage: fsck [-ANPRTV] [-C FD] [-t FSTYPE] [FS_OPTS] [BLOCKDEV]...

Check and repair filesystems

	-A	Walk /etc/fstab and check all filesystems
	-N	Don't execute, just show what would be done
	-P	With -A, check filesystems in parallel
	-R	With -A, skip the root filesystem
	-T	Don't show title on startup
	-V	Verbose
	-C n	Write status information to specified filedescriptor
	-t TYPE	List of filesystem types to check
```

---
[[BusyBox|← All BusyBox commands]]
