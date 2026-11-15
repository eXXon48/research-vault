---
applet: "xargs"
tags: [busybox/command]
---

# `xargs`

```text
Usage: xargs [OPTIONS] [PROG ARGS]

Run PROG on every item given by stdin

	-p	Ask user whether to run each command
	-r	Don't run command if input is empty
	-0	Input is separated by NUL characters
	-t	Print the command on stderr before execution
	-e[STR]	STR stops input processing
	-n N	Pass no more than N args to PROG
	-s N	Pass command line of no more than N bytes
	-x	Exit if size is exceeded
```

---
[[BusyBox|← All BusyBox commands]]
