---
applet: "mktemp"
tags: [busybox/command]
---

# `mktemp`

```text
Usage: mktemp [-dt] [-p DIR] [TEMPLATE]

Create a temporary file with name based on TEMPLATE and print its name.
TEMPLATE must end with XXXXXX (e.g. [/dir/]nameXXXXXX).
Without TEMPLATE, -t tmp.XXXXXX is assumed.

	-d	Make directory, not file
	-t	Prepend base directory name to TEMPLATE
	-p DIR	Use DIR as a base directory (implies -t)

Base directory is: -p DIR, else $TMPDIR, else /tmp
```

---
[[BusyBox|← All BusyBox commands]]
