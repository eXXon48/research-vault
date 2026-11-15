---
applet: "mknod"
tags: [busybox/command]
---

# `mknod`

```text
Usage: mknod [-m MODE] NAME TYPE MAJOR MINOR

Create a special file (block, character, or pipe)

	-m MODE	Creation mode (default a=rw)
TYPE:
	b	Block device
	c or u	Character device
	p	Named pipe (MAJOR and MINOR are ignored)
```

---
[[BusyBox|← All BusyBox commands]]
