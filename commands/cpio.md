---
applet: "cpio"
tags: [busybox/command]
---

# `cpio`

```text
Usage: cpio [-dmvu] [-F FILE] [-H newc] [-tio] [-p DIR] [EXTR_FILE]...

Extract or list files from a cpio archive, or
create an archive (-o) or copy files (-p) using file list on stdin

Main operation mode:
	-t	List
	-i	Extract EXTR_FILEs (or all)
	-o	Create (requires -H newc)
	-p DIR	Copy files to DIR
	-d	Make leading directories
	-m	Preserve mtime
	-v	Verbose
	-u	Overwrite
	-F FILE	Input (-t,-i,-p) or output (-o) file
	-H newc	Archive format
```

---
[[BusyBox|← All BusyBox commands]]
