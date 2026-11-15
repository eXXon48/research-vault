---
applet: "adjtimex"
tags: [busybox/command]
---

# `adjtimex`

```text
Usage: adjtimex [-q] [-o OFF] [-f FREQ] [-p TCONST] [-t TICK]

Read and optionally set system timebase parameters. See adjtimex(2)

	-q	Quiet
	-o OFF	Time offset, microseconds
	-f FREQ	Frequency adjust, integer kernel units (65536 is 1ppm)
		(positive values make clock run faster)
	-t TICK	Microseconds per tick, usually 10000
	-p TCONST
```

---
[[BusyBox|← All BusyBox commands]]
