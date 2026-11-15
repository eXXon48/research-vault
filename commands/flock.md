---
applet: "flock"
tags: [busybox/command]
---

# `flock`

```text
Usage: flock [-sxun] FD|{FILE [-c] PROG ARGS}

[Un]lock file descriptor, or lock FILE, run PROG

	-s	Shared lock
	-x	Exclusive lock (default)
	-u	Unlock FD
	-n	Fail rather than wait
```

---
[[BusyBox|← All BusyBox commands]]
