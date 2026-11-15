---
applet: "sed"
tags: [busybox/command]
---

# `sed`

```text
Usage: sed [-efinr] SED_CMD [FILE]...

	-e CMD	Add CMD to sed commands to be executed
	-f FILE	Add FILE contents to sed commands to be executed
	-i	Edit files in-place (else sends result to stdout)
	-n	Suppress automatic printing of pattern space
	-r	Use extended regex syntax

If no -e or -f, the first non-option argument is the sed command string.
Remaining arguments are input files (stdin if none).
```

---
[[BusyBox|← All BusyBox commands]]
