---
applet: "tail"
tags: [busybox/command]
---

# `tail`

```text
Usage: tail [OPTIONS] [FILE]...

Print last 10 lines of each FILE (or stdin) to stdout.
With more than one FILE, precede each with a filename header.

	-f		Print data as file grows
	-s SECONDS	Wait SECONDS between reads with -f
	-n N[kbm]	Print last N lines
	-c N[kbm]	Print last N bytes
	-q		Never print headers
	-v		Always print headers

N may be suffixed by k (x1024), b (x512), or m (x1024^2).
If N starts with a '+', output begins with the Nth item from the start
of each file, not from the end.
```

---
[[BusyBox|← All BusyBox commands]]
